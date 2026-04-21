# ReplicaSet Controller: Pod 생성과 조정 루프

## Deployment 적용부터 Pod 생성까지 — ReplicaSet Controller 내부 동작 분석

> * **대주제:** k8s 네트워크 스택 준비
> * **중주제:** ReplicaSet Controller

---

## 목차

1. 핵심 한 줄 요약
2. 사전 학습 요약 — client-go Informer & WorkQueue
3. Deployment → ReplicaSet → Pod 계층 구조
4. ReplicaSet Controller 초기화 — Informer 연결
5. `syncReplicaSet()` — Reconcile 루프
6. `slowStartBatch()` — 왜 한꺼번에 생성하지 않는가
7. 고아 파드 입양 & 삭제 대상 선정
8. 마무리
9. 부록: 주요 코드 경로

---

## 1. 핵심 한 줄 요약

> **"현재 Pod 수를 `spec.replicas`와 일치시키는 것 — 그것이 전부"**

* 부족하면 Pod 생성, 초과하면 Pod 삭제
* API 서버 상태가 바뀔 때마다 반복 수행 (Reconcile Loop)

| 하는 것                 | 하지 않는 것 (다른 컴포넌트 책임)                 |
|----------------------|--------------------------------------|
| Pod 수 조정 (생성/삭제)     | 노드 스케줄링 결정 (→ Scheduler)             |
| ReplicaSet 소유 Pod 추적 | 컨테이너 실행 관리 (→ Kubelet)               |
| 고아 Pod 입양/방출         | 롤링 업데이트 전략 (→ Deployment Controller) |

---

## 2. 사전 학습 요약 — client-go Informer & WorkQueue

* k8s의 모든 컨트롤러가 변경을 감지하고 반응하는 공통 기반
* ReplicaSet Controller도 이 패턴 위에서 동작함

### 컨트롤러의 위치

* ReplicaSet Controller는 `kube-controller-manager` 프로세스 안에서 실행됨
* Informer는 컨트롤러 내부에 포함 — **컨트롤러가 능동적으로 API 서버에 접속**

```
[kube-controller-manager 프로세스]

  ReplicaSet Controller
    │
    ├── Informer (ReplicaSet 감시) ──► API Server (List + Watch)
    │
    └── Informer (Pod 감시)        ──► API Server (List + Watch)
```

* ReplicaSet과 Pod, 두 오브젝트를 동시에 감시
* 어느 쪽이 변해도 Reconcile 트리거

### 핵심 흐름

```
[kube-controller-manager 프로세스]

  ┌─ SharedIndexInformer ──────────────────────────────┐
  │                                                    │
  │  Reflector (List + Watch) ◄── API Server           │
  │    │                                               │
  │    ▼                                               │
  │  DeltaFIFO ──► Indexer (local cache)               │
  │    │                                               │
  │    ▼                                               │
  │  EventHandler (Add / Update / Delete)              │
  │                                                    │
  └────────────────────────────────────────────────────┘
       │  key (namespace/name)
       ▼
     WorkQueue ──► worker goroutine ──► Reconcile()
```

* **List**: 시작 시 전체 오브젝트 목록 수신 → Indexer 초기화
* **Watch**: 이후 변경 이벤트만 스트리밍으로 수신 → Indexer 점진적 갱신
* Watch 연결 끊김 시 자동 재연결 → List부터 재시작

### 핵심 타입 (`k8s.io/client-go`)

| 패키지              | 타입                      | 역할                     |
|------------------|-------------------------|------------------------|
| `tools/cache`    | `SharedIndexInformer`   | List+Watch → 로컬 캐시 동기화 |
| `util/workqueue` | `RateLimitingInterface` | 속도 제한이 있는 작업 큐         |

#### `SharedIndexInformer`

* Shared: 같은 오브젝트 타입에 대해 여러 컨트롤러가 하나의 Informer 공유
    * kube-controller-manager 안의 수십 개 컨트롤러가 Pod Informer 하나를 공유
* Index: 로컬 캐시(Indexer)에서 인덱스 기반 빠른 조회 지원
* List+Watch + 캐시 관리 + 인덱싱을 모두 담당
* 내부 구성요소:
    * `Reflector` — List+Watch로 API 서버에서 이벤트 수신
    * `DeltaFIFO` — 변경 이벤트 버퍼 큐 (내부 구현)
        * API 서버 이벤트를 EventHandler로 전달 전 쌓아두는 큐
        * Delta: 이벤트 타입 5가지 — `Added`, `Updated`, `Deleted`, `Replaced`, `Sync`
        * FIFO: 선입선출 순서 보장
        * 같은 오브젝트의 이벤트가 여러 개 쌓이면 처리 전까지 병합 → 중복 처리 감소
    * `Indexer` — 로컬 캐시 + 인덱스 기반 조회

#### `RateLimitingInterface`

* worker goroutine이 처리할 작업 키(`namespace/name`) 큐
* 중복 제거: 같은 키가 큐에 이미 있으면 추가 삽입 안 함
* 재시도 속도 제한: Reconcile 실패 시 지수 백오프로 재삽입
* 디커플링: 이벤트 수신 속도와 처리 속도 분리

---

## 3. Deployment → ReplicaSet → Pod 계층 구조

* `kubectl apply -f deployment.yaml` 시 오브젝트 생성 순서

```
사용자
  │  kubectl apply
  ▼
API Server ──► Deployment 오브젝트 저장
                  │
                  ▼  (Deployment Controller가 감지)
              ReplicaSet 오브젝트 생성
                  │
                  ▼  (ReplicaSet Controller가 감지)
              Pod 오브젝트 생성
                  │
                  ▼  (kube-scheduler가 감지)
              Node에 바인딩 → kubelet이 컨테이너 실행
```

* 위 다이어그램의 각 "감지" 단계가 1절의 Informer 패턴으로 동작
    * Deployment Controller → Deployment Informer로 Watch → 변경 감지 시 ReplicaSet 생성/수정
    * ReplicaSet Controller → ReplicaSet Informer + Pod Informer로 Watch → 변경 감지 시 Pod 생성/삭제
    * kube-scheduler → Pod Informer로 Watch → nodeName 미지정 Pod 감지 시 노드 바인딩
* 각 컨트롤러는 자신의 입력 오브젝트를 Watch → 관심사 분리
* ReplicaSet Controller가 Informer를 2개 사용하는 이유
    * Reconcile의 핵심: desired(원하는 수) - actual(현재 수) = diff
    * ReplicaSet Informer: 원하는 파드 수(`spec.replicas`) 파악
    * Pod Informer: 현재 실제 파드 수 파악
    * 예시
        * `spec.replicas` 3→5 변경 → ReplicaSet Informer 감지 → Pod 2개 추가 생성
        * Pod 하나 소멸 → Pod Informer 감지 → Pod 1개 재생성
    * 어느 쪽이 변해도 Reconcile 트리거 → 두 Informer 모두 필요
* 각 레이어가 분리된 이유
    * 롤링 업데이트: Deployment가 구 ReplicaSet ↔ 신 ReplicaSet 비율을 점진적으로 조정
    * 단일 책임
        * ReplicaSet: "몇 개의 파드를 유지할 것인가"만 담당 — 버전이나 업데이트 전략에 관여하지 않음
        * Deployment: "어떤 버전으로, 어떤 전략으로 업데이트할 것인가" 담당
    * 예시: nginx 1.24 → 1.25 롤링 업데이트 (replicas=3, maxSurge=1, maxUnavailable=0)
        ```
        [시작 상태]
        Deployment (nginx:1.25 적용)
          ├── RS-old (nginx:1.24) replicas=3  →  Pod-A, Pod-B, Pod-C (Running)
          └── RS-new (nginx:1.25) replicas=0

        [1단계] Deployment Controller: RS-new replicas 0→1
          ├── RS-old replicas=3  →  Pod-A, Pod-B, Pod-C
          └── RS-new replicas=1  →  Pod-D (생성 중)
                                     └── ReplicaSet Controller가 Pod-D 생성

        [2단계] Pod-D Ready 확인 → Deployment Controller: RS-old replicas 3→2
          ├── RS-old replicas=2  →  Pod-A, Pod-B  (Pod-C 삭제)
          │                          └── ReplicaSet Controller가 Pod-C 삭제
          └── RS-new replicas=1  →  Pod-D

        [3단계~4단계] 반복 ...

        [완료]
          ├── RS-old replicas=0
          └── RS-new replicas=3  →  Pod-D, Pod-E, Pod-F
        ```
        * Deployment Controller: 각 단계에서 RS-old/RS-new의 replicas 수를 조정 — **전략 결정**
        * ReplicaSet Controller: replicas 값에 맞춰 Pod 생성/삭제 — **실행만 담당**
        * ReplicaSet Controller는 버전(nginx:1.24 vs 1.25)을 전혀 모름 — 숫자만 맞춤

---

## 4. ReplicaSet Controller 초기화 및 실행

### 전체 호출 경로

```
main()                                                           controller-manager.go:L34
  → NewControllerManagerCommand() → Run()                        controllermanager.go:L102, L185
      → NewControllerDescriptors()                               controller_descriptor.go:L148
          newReplicaSetControllerDescriptor()                    apps.go:L94
      → BuildControllers() → BuildController()                   controllermanager.go:L571
          → newReplicaSetController()                            apps.go:L102
              → NewReplicaSetController()                        replica_set.go:L155
                  → NewBaseController()                          replica_set.go:L204
                      rsc.syncHandler = rsc.syncReplicaSet       replica_set.go:L269
      → RunControllers() → controller.Run()                      controllermanager.go:L655
          → rsc.Run(ctx, workers)                                replica_set.go:L275
              → wait.UntilWithContext(rsc.worker, 1s)            replica_set.go:L300
                  → rsc.worker()                                 replica_set.go:L622
                      → rsc.processNextWorkItem()                replica_set.go:L627
                          → rsc.syncHandler() = syncReplicaSet() replica_set.go:L634
```

### 4.1 main() — kube-controller-manager 프로세스 진입

> `cmd/kube-controller-manager/controller-manager.go:L34`

```go
func main() {
    command := app.NewControllerManagerCommand()
    code := cli.Run(command)
    os.Exit(code)
}
```

> `cmd/kube-controller-manager/app/controllermanager.go:L102, L127~148`

```go
func NewControllerManagerCommand() *cobra.Command {
    // ...
    cmd := &cobra.Command{
        RunE: func(cmd *cobra.Command, args []string) error {
            // ...
            c, err := s.Config(ctx, KnownControllers(), ControllersDisabledByDefault(), ControllerAliases())
            // ...
            return Run(ctx, c.Complete())
        },
    }
}
```

* `main()` → `NewControllerManagerCommand()` → Cobra RunE 핸들러 → `Run(ctx, c.Complete())` 진입

### 4.2 컨트롤러 등록 및 생성

> `cmd/kube-controller-manager/app/controllermanager.go:L185, L248~283`

```go
func Run(ctx context.Context, c *config.CompletedConfig) error {
    // ...
    run := func(ctx context.Context, controllerDescriptors map[string]*ControllerDescriptor) {
        controllerContext, _ := CreateControllerContext(ctx, c, ...)
        controllers, _ := BuildControllers(ctx, controllerContext, controllerDescriptors, ...)
        controllerContext.InformerFactory.Start(stopCh)
        RunControllers(ctx, controllerContext, controllers, ...)
    }

    // No leader election, run directly
    controllerDescriptors := NewControllerDescriptors()
    run(ctx, controllerDescriptors)
}
```

* `// No leader election, run directly` — HA 환경에서 KCM(kube-controller-manager) 프로세스를 여러 개 띄우는 경우 리더 1개만 reconcile 실행(중복 처리·race 방지), 세미나에서는 단순화를 위해 leader election OFF 경로만 설명
    * ON 경로(기본값 `--leader-elect=true`): `leaderElectAndRun` 고루틴으로 경합 참여 → `OnStartedLeading` 콜백에서 `run(ctx, ...)` 실행, `OnStoppedLeading`에서 프로세스 종료
    * OFF 경로(`--leader-elect=false`): `run(ctx, ...)` 바로 호출 — 코드 블록에 발췌된 경로
    * 리더 결정: `kube-system` 네임스페이스의 Lease 오브젝트를 주기적으로 renew, 갱신 끊기면 다른 인스턴스가 낙관적 잠금으로 획득
* `NewControllerDescriptors()` — 모든 컨트롤러의 디스크립터(이름 + 생성자 함수)를 맵으로 수집 → 아래에서 상세
* `run` 클로저 내부:
    * `BuildControllers()` — 각 디스크립터의 생성자 함수를 호출해 컨트롤러 인스턴스 생성 → 아래에서 상세
    * `InformerFactory.Start()` — 모든 Informer의 List+Watch 시작 → 4.4에서 상세
    * `RunControllers()` — 각 컨트롤러를 별도 고루틴으로 실행 → 4.4에서 상세

**컨트롤러 디스크립터 등록**

> `cmd/kube-controller-manager/app/controller_descriptor.go:L148`

```go
func NewControllerDescriptors() map[string]*ControllerDescriptor {
    // ...
    register(newReplicaSetControllerDescriptor())
}
```

> `cmd/kube-controller-manager/app/apps.go:L94`

```go
func newReplicaSetControllerDescriptor() *ControllerDescriptor {
    return &ControllerDescriptor{
        name:        names.ReplicaSetController,  // "replicaset-controller"
        constructor: newReplicaSetController,
    }
}
```

* 각 컨트롤러를 이름 + 생성자 함수로 등록 — 실제 생성은 아직

**컨트롤러 인스턴스 생성**

> `cmd/kube-controller-manager/app/controllermanager.go:L248, L280`

```go
run := func(ctx context.Context, controllerDescriptors map[string]*ControllerDescriptor) {
    controllerContext, _ := CreateControllerContext(ctx, c, ...)
    controllers, _ := BuildControllers(ctx, controllerContext, controllerDescriptors, ...)
    // ...
}

// No leader election, run directly
controllerDescriptors := NewControllerDescriptors()
run(ctx, controllerDescriptors)
```

* `Run` 함수(L185)에서 leader election 분기 후 `run` 클로저 호출 — no-leader-election 경로에서 `NewControllerDescriptors()` 결과를 그대로 인자로 전달
* `run` 클로저 내부에서 `CreateControllerContext`로 `controllerContext` 생성 후 `BuildControllers` 호출 — 여기서 디스크립터 맵을 실제 인스턴스로 펼치는 단계 시작

**ControllerContext — 공유 자원 묶음**

> `cmd/kube-controller-manager/app/controllermanager.go:L408`

```go
type ControllerContext struct {
    ClientBuilder                   clientbuilder.ControllerClientBuilder
    InformerFactory                 informers.SharedInformerFactory
    ObjectOrMetadataInformerFactory informerfactory.InformerFactory
    ComponentConfig                 kubectrlmgrconfig.KubeControllerManagerConfiguration
    RESTMapper                      *restmapper.DeferredDiscoveryRESTMapper
    InformersStarted                chan struct{}
    ResyncPeriod                    func() time.Duration
    ControllerManagerMetrics        *controllersmetrics.ControllerManagerMetrics
    GraphBuilder                    *garbagecollector.GraphBuilder
}
```

모든 컨트롤러 생성자에 그대로 전달되는 컨테이너. `CreateControllerContext`(L475)에서 아래 순서로 채움 (RS 컨트롤러 관련 단계만):

1. `rootClientBuilder`로 shared-informers 전용 클라이언트 생성
2. `SharedInformerFactory` 생성 — 모든 컨트롤러가 공유하는 캐시
3. API 서버 최대 10초 대기
4. 필드 조립 후 반환

필드별 역할 (RS 컨트롤러가 실제 사용하는 필드):

| 필드              | 역할                                                               |
|-----------------|------------------------------------------------------------------|
| `ClientBuilder` | 컨트롤러별 전용 클라이언트 팩토리 — `NewClient("replicaset-controller")`로 호출    |
| `InformerFactory` | 공유 typed informer 팩토리 — RS·Pod informer를 여기서 꺼냄              |
| `ComponentConfig` | kube-controller-manager 전체 설정 — `ConcurrentRSSyncs` 등 컨트롤러별 값 보관 |

* `ControllerContext`는 ReplicaSet, Deployment, DaemonSet, GC 등 kube-controller-manager 내 모든 컨트롤러가 공유하는 컨테이너임. 여기서는 ReplicaSet Controller가 실제로 사용하는 필드만 정리함.

> `cmd/kube-controller-manager/app/controllermanager.go:L571, L578, L628`

```go
func BuildControllers(ctx context.Context, controllerCtx ControllerContext,
    controllerDescriptors map[string]*ControllerDescriptor, ...) ([]Controller, error) {

    buildController := func(controllerDesc *ControllerDescriptor) error {
        ctrl, err := controllerDesc.BuildController(ctx, controllerCtx)
        // ...
        controllers = append(controllers, ctrl)
        return nil
    }

    // SA Token Controller 먼저 빌드
    buildController(controllerDescriptors[names.ServiceAccountTokenController])

    for _, controllerDesc := range controllerDescriptors {
        if controllerDesc.RequiresSpecialHandling() { continue }
        if !controllerCtx.IsControllerEnabled(controllerDesc) { continue }
        buildController(controllerDesc)
    }
    return controllers, nil
}
```

* 디스크립터 맵을 순회하며 `buildController` 클로저로 각 컨트롤러 빌드
* SA Token Controller 우선 빌드 — 실패 시 즉시 에러 반환, 다른 컨트롤러 빌드 중단
  * 역할: 클러스터 전체 ServiceAccount의 토큰 Secret 생성·관리 — Pod가 마운트하는 `/var/run/secrets/kubernetes.io/serviceaccount/token`도 이 컨트롤러가 채움
  * 왜 먼저: 클러스터 필수 기능이므로 빌드 실패 시 fast-fail — 나머지 컨트롤러를 빌드하기 전에 실패를 조기에 감지하기 위함
* `RequiresSpecialHandling` 또는 비활성화된 컨트롤러는 skip — `RequiresSpecialHandling = true`는 현재 SA Token Controller 하나뿐, 이 플래그로 2단계 순회 시 중복 빌드 방지
* 빌드 결과 `Controller` 인터페이스를 `controllers` 슬라이스에 수집 → `RunControllers`에서 사용

// @TODO: 여기부터
> `cmd/kube-controller-manager/app/controller_descriptor.go:L93`

```go
func (r *ControllerDescriptor) BuildController(ctx context.Context, controllerCtx ControllerContext) (Controller, error) {
    // 요구 feature gate 비활성 시 nil 반환(skip)
    for _, featureGate := range r.GetRequiredFeatureGates() {
        if !utilfeature.DefaultFeatureGate.Enabled(featureGate) { return nil, nil }
    }
    // cloud provider controller skip
    if r.IsCloudProviderController() { return nil, nil }

    return r.GetControllerConstructor()(ctx, controllerCtx, controllerName)
}
```

* feature gate·cloud provider 조건 미충족 시 skip
* 통과 시 디스크립터에 등록된 `constructor` 필드(`newReplicaSetController`) 직접 호출 → 실제 구조체 생성 시작

호출 체인 요약:
`Run` → `run` 클로저 → `BuildControllers` → `buildController` 클로저 → `ControllerDescriptor.BuildController` → `newReplicaSetController`

> `cmd/kube-controller-manager/app/apps.go:L102`

```go
func newReplicaSetController(ctx context.Context, controllerContext ControllerContext, ...) (Controller, error) {
    client, _ := controllerContext.NewClient("replicaset-controller")
    rsc := replicaset.NewReplicaSetController(
        ctx,
        controllerContext.InformerFactory.Apps().V1().ReplicaSets(),
        controllerContext.InformerFactory.Core().V1().Pods(),
        client,
        replicaset.BurstReplicas,
    )
    return newControllerLoop(func(ctx context.Context) {
        rsc.Run(ctx, int(controllerContext.ComponentConfig.ReplicaSetController.ConcurrentRSSyncs))
    }, controllerName), nil
}
```

* InformerFactory에서 RS Informer, Pod Informer 주입
* `rsc.Run`을 `controllerLoop`으로 감싸 `Controller` 인터페이스로 반환

### 4.3 NewReplicaSetController() / NewBaseController() — 구조체 생성

> `pkg/controller/replicaset/replica_set.go:L155, L204`

구조체 생성 시 핵심 필드 초기화:

| 필드                       | 역할                                               |
|--------------------------|--------------------------------------------------|
| `kubeClient`             | Pod 생성/삭제 등 쓰기 시 API 서버 직접 호출                    |
| `podControl`             | `kubeClient` 래퍼 — 테스트 시 mock 교체 가능, 이벤트 기록 일괄 처리 |
| `expectations`           | 진행 중 작업 추적 — 캐시 갱신 전 중복 reconcile 방지             |
| `queue`                  | `RateLimitingInterface` — 2절에서 설명한 WorkQueue     |
| `burstReplicas`          | 한 번의 reconcile에서 최대 500 Pod 처리 (바깥 한계)           |
| `rsLister` / `podLister` | 로컬 캐시(Indexer) 읽기 전용 — API 서버 호출 없이 메모리 즉시 조회    |

* `rsLister` / `podLister`로 읽고, 쓰기(생성/삭제)는 반드시 `kubeClient`/`podControl`로 API 서버 직접 요청
* `expectations` 상세:
    * 문제: Pod 생성 요청 후 캐시 갱신 전에 Reconcile이 재트리거되면 중복 생성 위험
    * 생성(add): 단순 카운터 — Pod Add 이벤트마다 1 차감
    * 삭제(del): UID 목록 추적 — 삭제 요청한 특정 Pod의 UID만 매칭 (단순 카운터면 다른 Pod 크래시가 잘못 차감 가능)
    * TTL: 응답 없어도 만료 후 Reconcile 재개 → 무한 블로킹 방지
* `burstReplicas`와 `slowStartBatch` 관계:
    * `burstReplicas`: 한 번의 Reconcile에서의 최대 수 제한 (바깥 한계)
    * `slowStartBatch`: 그 안에서 1→2→4→8 배치로 점진적 생성 (안의 속도 조절) — 6절에서 상세 설명

**핵심 와이어링**

> `pkg/controller/replicaset/replica_set.go:L269`

```go
rsc.syncHandler = rsc.syncReplicaSet
```

* `processNextWorkItem`에서 `rsc.syncHandler`를 호출 → 실제 구현인 `syncReplicaSet`으로 연결

**Informer 이벤트 핸들러 등록**

> `pkg/controller/replicaset/replica_set.go:L207, L223~233, L251~264`

```go
rsc := &ReplicaSetController{ ... }

rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    func(obj interface{}) { rsc.addRS(logger, obj) },
    UpdateFunc: func(oldObj, newObj interface{}) { rsc.updateRS(logger, oldObj, newObj) },
    DeleteFunc: func(obj interface{}) { rsc.deleteRS(logger, obj) },
})

podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    func(obj interface{}) { rsc.addPod(logger, obj) },
    UpdateFunc: func(oldObj, newObj interface{}) { rsc.updatePod(logger, oldObj, newObj) },
    DeleteFunc: func(obj interface{}) { rsc.deletePod(logger, obj) },
})
```

* RS Informer와 Pod Informer 모두 이 큐에 **RS 키**를 삽입 — Reconcile 단위는 항상 RS
* RS Informer EventHandler: RS 키를 queue에 삽입 → 비동기 처리
* Pod Informer EventHandler: expectations 업데이트(즉시) + Pod의 owner RS 키를 queue에 삽입

##### 유틸 함수: `enqueueRS`

> `pkg/controller/replicaset/replica_set.go:L367~375`

```go
func (rsc *ReplicaSetController) enqueueRS(rs *apps.ReplicaSet) {
    key, err := controller.KeyFunc(rs)
    if err != nil { ... }
    rsc.queue.Add(key)
}
```

* RS 오브젝트 → 키 추출 → `queue.Add(key)` 래퍼
* 코드에서 `enqueueRS(rs)`와 `queue.Add(rsKey)` 혼용
    * 입력이 RS 오브젝트면 `enqueueRS`, 키를 이미 갖고 있으면 `queue.Add` 직접 호출
    * 결과는 동일

##### RS Informer 핸들러

> `pkg/controller/replicaset/replica_set.go:L223~233`

```go
rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    func(obj interface{}) { rsc.addRS(logger, obj) },
    UpdateFunc: func(oldObj, newObj interface{}) { rsc.updateRS(logger, oldObj, newObj) },
    DeleteFunc: func(obj interface{}) { rsc.deleteRS(logger, obj) },
})
```

**`addRS()`**

> `pkg/controller/replicaset/replica_set.go:L387~391`

```go
func (rsc *ReplicaSetController) addRS(logger klog.Logger, obj interface{}) {
    rs := obj.(*apps.ReplicaSet)
    rsc.enqueueRS(rs)
}
```

* RS 키를 queue에 삽입 — 단순

**`updateRS()`**

> `pkg/controller/replicaset/replica_set.go:L399~409, L426`

```go
if curRS.UID != oldRS.UID {
    rsc.deleteRS(logger, cache.DeletedFinalStateUnknown{Key: key, Obj: oldRS})
}
rsc.enqueueRS(curRS)
```

* RS 키를 queue에 삽입 (기본 동작)
* `spec.replicas` 변경 외에도 Update 이벤트 발생 시 무조건 enqueue
    * status 업데이트로 불필요한 Reconcile이 일부 발생하지만 로컬 캐시만 읽으므로 부하 없음
    * Pod 생성 실패로 멈춘 RS를 Informer 전체 재동기화 시 복구 가능
* UID 변경 감지 (같은 이름으로 RS 재생성된 경우)
    * Informer가 Delete + Add 대신 하나의 Update로 합칠 수 있는 알려진 한계 (소스 TODO 주석 확인)
    * `curRS.UID != oldRS.UID` → `deleteRS(oldRS)` 호출로 이전 RS의 stale expectations 강제 삭제
    * 이 처리 없으면 새 RS가 이전 RS의 미충족 expectations를 물려받아 영구 블로킹 위험
    * `oldRS`를 `cache.DeletedFinalStateUnknown`으로 감싸 전달 — `deleteRS`의 tombstone 처리 코드 재사용

**`deleteRS()`**

> `pkg/controller/replicaset/replica_set.go:L429~459`

```go
// tombstone 처리 — k8s 컨트롤러 Delete 핸들러 표준 패턴
rs, ok := obj.(*apps.ReplicaSet)
if !ok {
    tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
    if !ok { ... }
    rs, ok = tombstone.Obj.(*apps.ReplicaSet)
    if !ok { ... }
}

// ...
rsc.expectations.DeleteExpectations(logger, key)
rsc.queue.Add(key)
```

* **tombstone (`DeletedFinalStateUnknown`) 2단계 type assertion** — k8s 컨트롤러 Delete 핸들러 표준 패턴
    * 1단계: `obj`를 `*apps.ReplicaSet`으로 직접 assertion → 정상 Delete 이벤트
    * 2단계: 실패 시 `cache.DeletedFinalStateUnknown`으로 assertion → tombstone에서 `.Obj` 추출
* **tombstone이란** — Watch 연결 끊김 후 Relist 시 DeltaFIFO가 합성하는 가짜 Delete 이벤트 래퍼
    * Watch가 끊긴 사이에 삭제된 오브젝트 → Delete 이벤트 미수신
    * Relist로 받은 새 목록에 해당 키 없음 → DeltaFIFO가 "삭제됐음"을 추론
    * 마지막으로 알려진(stale할 수 있는) 오브젝트를 `DeletedFinalStateUnknown{Key, Obj}`로 감싸 전달
    * 정의: `staging/src/k8s.io/client-go/tools/cache/delta_fifo.go:L793~800`
* `expectations.DeleteExpectations(key)` — 해당 RS의 expectations 전부 삭제
* `queue.Add(key)` — Reconcile 기회 부여 (이미 삭제됐으면 즉시 종료)

##### Pod Informer 핸들러

> `pkg/controller/replicaset/replica_set.go:L251~264`

```go
podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    func(obj interface{}) { rsc.addPod(logger, obj) },
    UpdateFunc: func(oldObj, newObj interface{}) { rsc.updatePod(logger, oldObj, newObj) },
    DeleteFunc: func(obj interface{}) { rsc.deletePod(logger, obj) },
})
```

**`addPod()`**

> `pkg/controller/replicaset/replica_set.go:L466~501`

```go
if pod.DeletionTimestamp != nil {
    rsc.deletePod(logger, pod)
    return
}
if controllerRef := metav1.GetControllerOf(pod); controllerRef != nil {
    rs := rsc.resolveControllerRef(pod.Namespace, controllerRef)
    rsc.expectations.CreationObserved(logger, rsKey)
    rsc.queue.Add(rsKey)
    return
}
// 고아 Pod — 매칭 RS 전부 enqueue
rss := rsc.getPodReplicaSets(pod)
for _, rs := range rss {
    rsc.enqueueRS(rs)
}
```

* **`DeletionTimestamp` 있는 경우** → `deletePod()` 위임 (컨트롤러 재시작 시 이미 삭제 중인 Pod가 보일 수 있음)
* **ControllerRef 있는 경우** (일반)
    * `expectations.CreationObserved(rsKey)` — add 카운터 즉시 차감
    * owner RS 키를 queue에 삽입
* **ControllerRef 없는 고아 Pod**
    * 셀렉터 매칭 RS 전부 enqueue — 입양 여부 판단은 `syncReplicaSet`에서 수행
    * expectations 차감 안 함 — 고아 Pod 생성을 기다리는 컨트롤러 없음

**`deletePod()`**

> `pkg/controller/replicaset/replica_set.go:L601~617`

```go
controllerRef := metav1.GetControllerOf(pod)
if controllerRef == nil {
    return  // 고아 Pod 무시
}
rs := rsc.resolveControllerRef(pod.Namespace, controllerRef)
rsc.expectations.DeletionObserved(logger, rsKey, controller.PodKey(pod))
rsc.queue.Add(rsKey)
```

* ControllerRef 없는 고아 Pod → 무시
* `expectations.DeletionObserved(rsKey, podUID)` — del expectations에서 해당 UID 제거 (즉시)
* owner RS 키를 queue에 삽입
* Delete 이벤트 유실 시 tombstone(`DeletedFinalStateUnknown`) 처리 포함

**`updatePod()`**

> `pkg/controller/replicaset/replica_set.go:L509~577`

```go
if curPod.ResourceVersion == oldPod.ResourceVersion { return }

if curPod.DeletionTimestamp != nil {
    rsc.deletePod(logger, curPod)
    if labelChanged { rsc.deletePod(logger, oldPod) }
    return
}
if controllerRefChanged && oldControllerRef != nil {
    if rs := rsc.resolveControllerRef(oldPod.Namespace, oldControllerRef); rs != nil {
        rsc.enqueueRS(rs)
    }
}
if curControllerRef != nil {
    rs := rsc.resolveControllerRef(curPod.Namespace, curControllerRef)
    rsc.enqueueRS(rs)
    return
}
if labelChanged || controllerRefChanged {
    rss := rsc.getPodReplicaSets(curPod)
    for _, rs := range rss { rsc.enqueueRS(rs) }
}
```

* `ResourceVersion` 동일 → 무시 (Informer 주기적 재동기화에 의한 중복 이벤트)
* **`DeletionTimestamp` 설정됨** → `deletePod()` 위임
    * graceful 삭제 시 실제 삭제까지 기다리지 않고 `DeletionTimestamp` 설정 시점에 즉시 삭제로 처리 → RS가 빠르게 대체 Pod 생성
    * 레이블도 변경된 경우 old/cur 모두 `deletePod()` 호출
* **ControllerRef 변경됨** → 이전 owner RS도 enqueue (소유권 이전 시 양쪽 RS Reconcile)
* **ControllerRef 있음** (일반) → owner RS enqueue
* **고아 Pod의 레이블/ControllerRef 변경** → 매칭 RS 전부 enqueue (입양 가능성 확인)

### 4.4 Run() — 워커 시작

> `pkg/controller/replicaset/replica_set.go:L275`

```go
func (rsc *ReplicaSetController) Run(ctx context.Context, workers int) {
    rsc.eventBroadcaster.StartStructuredLogging(3)
    rsc.eventBroadcaster.StartRecordingToSink(...)

    if !cache.WaitForNamedCacheSyncWithContext(ctx, rsc.podListerSynced, rsc.rsListerSynced) {
        return
    }

    for i := 0; i < workers; i++ {
        wg.Go(func() {
            wait.UntilWithContext(ctx, rsc.worker, time.Second)
        })
    }
    <-ctx.Done()
}
```

* 이벤트 브로드캐스터 시작
* `WaitForNamedCacheSyncWithContext` — RS/Pod Informer 캐시가 모두 동기화될 때까지 블로킹
    * 캐시 미동기화 상태에서 Reconcile 시작 시 stale 데이터로 잘못된 Pod 수 계산 위험
* `workers`(기본값: `ConcurrentRSSyncs`) 개의 고루틴 시작
    * `wait.UntilWithContext(rsc.worker, 1s)` — worker 패닉/종료 시 1초 후 자동 재시작

### 4.5 worker() → processNextWorkItem() — 작업 루프

> `pkg/controller/replicaset/replica_set.go:L622, L627`

```go
func (rsc *ReplicaSetController) worker(ctx context.Context) {
    for rsc.processNextWorkItem(ctx) {
    }
}

func (rsc *ReplicaSetController) processNextWorkItem(ctx context.Context) bool {
    key, quit := rsc.queue.Get()       // 블로킹 대기
    if quit { return false }
    defer rsc.queue.Done(key)

    err := rsc.syncHandler(ctx, key)   // = syncReplicaSet()
    if err == nil {
        rsc.queue.Forget(key)          // 성공: rate limiter 초기화
        return true
    }
    rsc.queue.AddRateLimited(key)      // 실패: 지수 백오프 후 재삽입
    return true
}
```

* `worker`: `processNextWorkItem()`을 큐 종료 시까지 무한 반복
* `queue.Get()` — 항목이 없으면 블로킹, 있으면 즉시 반환
* `rsc.syncHandler` = `rsc.syncReplicaSet` (4.3의 와이어링) → 다음 섹션으로 연결

---

## 5. `syncReplicaSet()` — Reconcile 루프

* WorkQueue에서 꺼낸 키(namespace/name)로 실제 조정을 수행하는 핵심 함수

### 처리 흐름 개요

```
// pkg/controller/replicaset/replica_set.go:L755
syncReplicaSet(key)
  │
  ├─ ReplicaSet 조회 (rsLister)
  ├─ 해당 RS 소유 Pod 목록 수집 (getPodsForReplicaSet)
  ├─ active / terminating Pod 분리
  │
  ├─ expectations 만족 여부 확인
  │     └─ 미충족 시 manageReplicas() 건너뜀
  │
  ├─ manageReplicas()
  │     ├─ diff = desired - active 계산
  │     ├─ diff > 0 → slowStartBatch()로 Pod 생성
  │     └─ diff < 0 → getPodsToDelete()로 Pod 삭제
  │
  └─ 상태 업데이트 (updateReplicaSetStatus)
```

### 상세 코드 분석

#### 1단계 — RS 조회

> `pkg/controller/replicaset/replica_set.go:L778~787`

```go
rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name)
if apierrors.IsNotFound(err) {
    rsc.expectations.DeleteExpectations(logger, key)
    return nil
}
```

* 로컬 캐시에서 RS 조회 — API 서버 호출 없음
* `IsNotFound` → RS 이미 삭제됨 → expectations 정리 후 정상 종료
    * `nil` 반환으로 worker가 `queue.Forget(key)` 호출 → 재시도 카운터 초기화
* 조회한 RS는 "Informer가 마지막으로 수신한 스냅샷" — 실제 현재 상태와 약간의 차이 가능

##### 비고 — queue.Forget 동작

`RateLimitingInterface`는 두 개의 독립된 상태를 관리함

| 레이어          | 관련 메서드                               | 추적 내용                             |
|--------------|--------------------------------------|-----------------------------------|
| Base Queue   | `Get()`, `Done(key)`                 | 현재 처리 중인 키 (`processing` 집합)      |
| Rate Limiter | `AddRateLimited(key)`, `Forget(key)` | 키별 실패 횟수 (`failures map[key]int`) |

`processNextWorkItem` 흐름:

```go
key, quit := rsc.queue.Get()       // processing 집합에 추가
defer rsc.queue.Done(key)          // 항상: processing에서 제거

err := rsc.syncHandler(ctx, key)
if err == nil {
    rsc.queue.Forget(key)          // 성공 시: rate limiter 실패 카운터 삭제
    return true
}
rsc.queue.AddRateLimited(key)      // 실패 시: backoff delay 후 큐 재삽입
return true
```

* `Forget(key)` = rate limiter의 `delete(failures, key)` — 큐에서 제거하는 게 아님
* 기본 backoff: `baseDelay(5ms) * 2^실패횟수`, 최대 1000s
    * `Forget` 없이 성공하면 `failures` 맵에 이전 실패 횟수가 남음 → 다음 실패 시 누적된 backoff 적용
    * `Forget` 호출 → 카운터 초기화 → 다음 실패부터 5ms로 재시작
* `Done`과 `Forget`은 완전히 다른 레이어 — `Forget`만 호출하고 `Done` 생략 시 해당 키가 `processing`에 남아 큐에서 재처리 불가

#### 2단계 — Pod 목록 수집 & 분류

> `pkg/controller/replicaset/replica_set.go:L797~813`

```go
allRSPods, err := controller.FilterPodsByOwner(rsc.podIndexer, &rs.ObjectMeta, rsc.Kind, true)

allActivePods := controller.FilterActivePods(logger, allRSPods)
activePods, err := rsc.claimPods(ctx, rs, selector, allActivePods)

allTerminatingPods := controller.FilterTerminatingPods(allRSPods)
terminatingPods := controller.FilterClaimedPods(rs, selector, allTerminatingPods)
```

* `FilterPodsByOwner`: RS UID로 인덱싱된 Pod + 셀렉터 매칭 고아 Pod 전부 수집
* `FilterActivePods`: Succeeded / Failed / 이미 삭제된 Pod 제외 → Active Pod만 추출
* `claimPods`: Active Pod 중 셀렉터 매칭 확인 → 고아 Pod 입양 → 최종 `activePods` 반환
    * 입양 성공한 고아 Pod도 `activePods`에 포함 → 이후 diff 계산에 반영

**Terminating Pod 분리** (feature gate `DeploymentReplicaSetTerminatingReplicas`)
* `DeletionTimestamp` 설정된 Pod를 Active에서 분리
* `manageReplicas`의 active 카운트에 포함 안 됨 → Terminating Pod를 active로 보면 diff = 0으로 판단해 대체 Pod 생성 안 됨 → 분리함으로써 즉시 대체 Pod 생성 가능
* status 계산에는 별도로 전달 → `TerminatingReplicas` 필드 반영

#### 3단계 — expectations 확인 & manageReplicas 호출 조건

> `pkg/controller/replicaset/replica_set.go:L789, L818~820`

```go
rsNeedsSync := rsc.expectations.SatisfiedExpectations(logger, key)
...
if rsNeedsSync && rs.DeletionTimestamp == nil {
    manageReplicasErr = rsc.manageReplicas(ctx, activePods, rs)
}
```

* `rsNeedsSync = true` 조건 (둘 중 하나)
    * add/del expectations가 모두 0 — 미충족 작업 없음
    * expectations TTL 만료 — 무한 블로킹 방지
* `rs.DeletionTimestamp == nil` 조건
    * RS 삭제 중 → 새 Pod 생성 불필요, 삭제 대상 Pod는 GC에 위임
* 두 조건 중 하나라도 불만족 → `manageReplicas` 건너뜀 → 상태 업데이트만 수행

#### 4단계 — manageReplicas: diff 계산 및 분기

> `pkg/controller/replicaset/replica_set.go:L650`

```go
diff := len(activePods) - int(*(rs.Spec.Replicas))
```

* `diff < 0`: 파드 부족 → 생성 경로
* `diff > 0`: 파드 과잉 → 삭제 경로
* `diff == 0`: 아무 작업 없음

**생성 경로 (diff < 0)**

> `pkg/controller/replicaset/replica_set.go:L657~698`

```go
diff = min(-diff, rsc.burstReplicas)
rsc.expectations.ExpectCreations(logger, rsKey, diff)
successfulCreations, err := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() error {
    return rsc.podControl.CreatePods(...)
})
// slowStartBatch 중단으로 시도 못 한 Pod 수만큼 expectations 즉시 차감
for i := 0; i < diff-successfulCreations; i++ {
    rsc.expectations.CreationObserved(logger, rsKey)
}
```

* `burstReplicas` 상한 적용 (기본 500)
* `ExpectCreations(diff)` **먼저** 등록 → 이후 Pod Add 이벤트가 차감
* 배치 중단 시 건너뛴 수만큼 즉시 차감 → Informer 이벤트 없이 선제 차감하지 않으면 블로킹

**삭제 경로 (diff > 0)**

> `pkg/controller/replicaset/replica_set.go:L700~747`

```go
diff = min(diff, rsc.burstReplicas)
podsToDelete := getPodsToDelete(activePods, relatedPods, diff)
rsc.expectations.ExpectDeletions(logger, rsKey, getPodKeys(podsToDelete))
var wg sync.WaitGroup
wg.Add(diff)
for _, pod := range podsToDelete {
    go func(targetPod *v1.Pod) {
        defer wg.Done()
        if err := rsc.podControl.DeletePod(...); err != nil {
            rsc.expectations.DeletionObserved(logger, rsKey, podKey)  // 실패 시 즉시 차감
        }
    }(pod)
}
wg.Wait()
```

* `ExpectDeletions` **먼저** 등록 (UID 목록) → Pod Delete 이벤트가 차감
* 삭제는 **병렬 goroutine** — 각 삭제 독립적, 실패해도 다른 삭제에 영향 없음
* 삭제 API 실패 시 즉시 차감 → 다음 Reconcile에서 재시도

**생성 vs 삭제 비대칭 설계**

|       | 생성                                       | 삭제                       |
|-------|------------------------------------------|--------------------------|
| 실행 방식 | `slowStartBatch` — 순차 배치, 오류 시 중단        | 병렬 goroutine, 오류 시 개별 처리 |
| 이유    | 실패 원인(쿼터 부족 등)이 동일 → 전부 시도 전에 빠르게 중단이 유리 | 실패가 독립적 → 병렬 처리로 지연 없음   |

#### 5단계 — 상태 업데이트

> `pkg/controller/replicaset/replica_set.go:L824~831`

```go
newStatus := calculateStatus(rs, activePods, terminatingPods, manageReplicasErr, ...)
updatedRS, err := updateReplicaSetStatus(logger, rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace), rs, newStatus, ...)
```

* `manageReplicas` 오류 여부와 무관하게 **항상** 상태 업데이트 실행
* `calculateStatus`: `ReadyReplicas`, `AvailableReplicas`, `TerminatingReplicas` 등 계산
* `updateReplicaSetStatus`: API 서버에 status 패치 (로컬 캐시가 아닌 실제 저장)
* 오류 순서: 상태 업데이트 오류 → 즉시 반환 / `manageReplicas` 오류 → 상태 업데이트 후 반환

#### 6단계 — MinReadySeconds 재동기화 예약

> `pkg/controller/replicaset/replica_set.go:L844~855`

```go
if updatedRS.Spec.MinReadySeconds > 0 &&
    updatedRS.Status.ReadyReplicas != updatedRS.Status.AvailableReplicas {
    nextSyncDuration = ...
    rsc.queue.AddAfter(key, *nextSyncDuration)
}
```

* `MinReadySeconds`: Pod가 Ready 상태로 N초 유지돼야 Available로 간주
* Ready지만 아직 Available 아닌 Pod 존재 → N초 후 재동기화 예약
* 예약 없으면 Available 전환 시 Informer 이벤트가 없어 status 업데이트 누락 가능

---

## 6. `slowStartBatch()` — 왜 한꺼번에 생성하지 않는가

* Pod를 1→2→4→8 배치로 점진적으로 생성하는 메커니즘

### 배치 성장 패턴

```
initialCount = 1

배치 1: 1개 시도  → 성공 → 다음 배치 크기 2배
배치 2: 2개 시도  → 성공 → 다음 배치 크기 2배
배치 3: 4개 시도  → 성공 → ...
배치 N: 남은 수 전부 시도
```

* 어느 배치에서든 오류 발생 시 이후 배치 모두 중단
* 설계 의도: API 서버 부하 제어 — 한꺼번에 수백 개 요청 시 API 서버 과부하 방지

* (상세 코드 분석 예정)

---

## 7. 고아 파드 입양 & 삭제 대상 선정

### 고아 파드 입양 (Orphan Adoption)

* 고아 파드: `ownerReferences`가 없거나 owner가 사라진 파드
* `getPodsForReplicaSet()` → `claimPods()` 흐름
    * 셀렉터 매칭 확인
    * ControllerRef 패치로 소유권 등록

```
ownerReferences 없는 파드
  + 레이블이 RS 셀렉터와 일치
  → ControllerRef 패치 → RS의 소유 파드로 편입
```

### 삭제 대상 선정 우선순위

* 스케일 다운 시 어떤 파드를 먼저 삭제할지 결정
* `getPodsToDelete()` 내 정렬 기준

| 우선순위 (높을수록 먼저 삭제) | 조건                     |
|-------------------|------------------------|
| 1                 | 아직 노드에 스케줄되지 않은 파드     |
| 2                 | Pending 상태 파드          |
| 3                 | Ready 아닌 파드            |
| 4                 | 같은 노드에 동일 RS 파드가 많은 경우 |
| 5                 | Running / Ready 파드     |

* (상세 코드 분석 예정)

---

## 8. 마무리

### ReplicaSet Controller = k8s 컨트롤러 패턴의 교과서

* Informer → EventHandler → WorkQueue → Reconcile 패턴이 그대로 드러남
* 단일 책임: "현재 파드 수 = 원하는 파드 수" 만 유지
* expectations 로 이중 조정 방지, slowStartBatch 로 API 부하 분산

### 다음 세미나 예고 — 0006 kube-proxy iptables

* 같은 Informer + Reconcile 패턴이 **네트워크 레이어**에 적용됨
* kube-proxy는 Service / EndpointSlice 변경을 감지 → iptables 규칙 갱신
* "ClusterIP로 패킷을 보내면 어떻게 Pod에 도달하는가"

---

## 9. 부록: 주요 코드 경로

| 항목            | 경로                                                                         |
|---------------|----------------------------------------------------------------------------|
| 컨트롤러 메인       | `pkg/controller/replicaset/replica_set.go`                                 |
| 유틸리티          | `pkg/controller/replicaset/replica_set_utils.go`                           |
| Pod 생성/삭제 추상화 | `pkg/controller/controller_utils.go`                                       |
| expectations  | `pkg/controller/controller_utils.go` (`UIDTrackingControllerExpectations`) |
| WorkQueue     | `vendor/k8s.io/client-go/util/workqueue`                                   |

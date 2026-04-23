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

| 필드                | 역할                                                                    |
|-------------------|-----------------------------------------------------------------------|
| `ClientBuilder`   | 컨트롤러별 전용 k8s API 클라이언트 팩토리 — `NewClient("replicaset-controller")`로 호출 |
| `InformerFactory` | 공유 typed informer 팩토리 — RS·Pod informer를 여기서 꺼냄                       |
| `ComponentConfig` | kube-controller-manager 전체 설정 — `ConcurrentRSSyncs` 등 컨트롤러별 값 보관      |

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

> `cmd/kube-controller-manager/app/controller_descriptor.go:L93`

```go
func (r *ControllerDescriptor) BuildController(ctx context.Context, controllerCtx ControllerContext) (Controller, error) {
    logger := klog.FromContext(ctx)
    controllerName := r.Name()

    // 요구 feature gate 비활성 시 nil 반환(skip)
    for _, featureGate := range r.GetRequiredFeatureGates() {
        if !utilfeature.DefaultFeatureGate.Enabled(featureGate) { return nil, nil }
    }
    // cloud provider controller skip
    if r.IsCloudProviderController() { return nil, nil }

    ctx = klog.NewContext(ctx, klog.LoggerWithName(logger, controllerName))
    return r.GetControllerConstructor()(ctx, controllerCtx, controllerName)
}
```

* feature gate·cloud provider 조건 미충족 시 skip
* 통과 시 디스크립터에 등록된 `constructor` 필드(`newReplicaSetController`) 직접 호출 → 실제 구조체 생성 시작

호출 체인 요약:
`Run` → `run` 클로저 → `BuildControllers` → `buildController` 클로저 → `ControllerDescriptor.BuildController` → `newReplicaSetController`

> `cmd/kube-controller-manager/app/apps.go:L102`

```go
func newReplicaSetController(ctx context.Context, controllerContext ControllerContext, controllerName string) (Controller, error) {
    client, err := controllerContext.NewClient("replicaset-controller")
    if err != nil {
        return nil, err
    }
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
* `rsc.Run`을 `controllerLoop`으로 감싸 `Controller` 인터페이스로 반환 (`Name`, `Run` 메소드 구현)

### 4.3 NewReplicaSetController() / NewBaseController() — 구조체 생성

**`NewReplicaSetController()`**

> `pkg/controller/replicaset/replica_set.go:L155`

```go
func NewReplicaSetController(ctx context.Context, rsInformer appsinformers.ReplicaSetInformer,
    podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int) *ReplicaSetController {
    eventBroadcaster := record.NewBroadcaster(record.WithContext(ctx))
    // consistencyStore — StaleControllerConsistencyReplicaSet feature gate 분기 (생략)
    return NewBaseController(logger, rsInformer, podInformer, kubeClient, burstReplicas,
        apps.SchemeGroupVersion.WithKind("ReplicaSet"),
        "replicaset_controller",
        "replicaset",
        controller.RealPodControl{
            KubeClient: kubeClient,
            Recorder:   eventBroadcaster.NewRecorder(...),
            OnWrite:    podWriteCallback,
        },
        eventBroadcaster,
        DefaultReplicaSetControllerFeatures(),
        consistencyStore,
    )
}
```

* ReplicaSet 전용 설정을 조립해서 `NewBaseController`로 위임하는 얇은 래퍼
* `NewBaseController`는 `ReplicationController`(`pkg/controller/replication`)와 공유하는 공통 구현
    * **ReplicationController란** — ReplicaSet의 전신(`v1.ReplicationController`). "원하는 수의 Pod 유지"라는 역할은 동일하지만 등호 셀렉터(`=`)만 지원하고 집합 기반 셀렉터(`in`, `notin`)를 지원하지 않는 제한이 있어 ReplicaSet으로 대체됨. 현재는 하위 호환 목적으로 잔존
    * **왜 공유하는가** — 두 컨트롤러의 reconcile 로직(Informer 핸들러, WorkQueue, syncHandler, expectations 관리)이 동일. 달라지는 건 "어떤 리소스 타입을 다루는가"뿐이므로 공통 로직을 `NewBaseController`로 추출하고 리소스 타입 정보(GVK, metric 이름, queueName)를 파라미터로 주입
        * **GVK(`GroupVersionKind`)란** — Kubernetes API 타입을 유일하게 식별하는 3-tuple. `apps/v1/ReplicaSet` vs `v1/ReplicationController`(core group, Group은 빈 문자열). Pod 생성 시 `ownerReference`의 Kind 필드 설정에 사용
* `RealPodControl` — `PodControlInterface`(`pkg/controller/controller_utils.go:L470`)의 실제 구현체. Pod 생성(`CreatePods`)/삭제(`DeletePod`) API 호출 + `Recorder`로 이벤트 기록. 테스트 코드에서는 `FakePodControl`로 교체 가능
* `consistencyStore` — `StaleControllerConsistencyReplicaSet` feature gate 활성 시 Pod 쓰기의 ResourceVersion 추적 → Informer 캐시가 아직 반영하지 못한 stale 데이터 기반 Reconcile 방지
* `eventBroadcaster` — 컨트롤러 이벤트(Pod 생성/삭제 성공·실패 등)를 API 서버에 기록하고 구조화 로깅

**`NewBaseController()` — 구조체 초기화**

> `pkg/controller/replicaset/replica_set.go:L204`

```go
rsc := &ReplicaSetController{
    GroupVersionKind: gvk,
    kubeClient:       kubeClient,
    podControl:       podControl,
    eventBroadcaster: eventBroadcaster,
    burstReplicas:    burstReplicas,
    expectations:     controller.NewUIDTrackingControllerExpectations(controller.NewControllerExpectations()),
    queue: workqueue.NewTypedRateLimitingQueueWithConfig(
        workqueue.DefaultTypedControllerRateLimiter[string](),
        workqueue.TypedRateLimitingQueueConfig[string]{Name: queueName},
    ),
    clock:              clock.RealClock{},
    controllerFeatures: controllerFeatures,
    consistencyStore:   consistencyStore,
}
```

| 필드              | 역할                                                                        |
|-----------------|---------------------------------------------------------------------------|
| `kubeClient`    | Pod 생성/삭제 등 API 서버 직접 호출                                                  |
| `podControl`    | `RealPodControl` — `kubeClient` 래퍼, 이벤트 기록 포함·테스트 시 mock 교체 가능            |
| `burstReplicas` | 한 번의 Reconcile에서 최대 생성/삭제 수 (기본 500)                                      |
| `expectations`  | `UIDTrackingControllerExpectations` — 진행 중 작업 추적, 캐시 갱신 전 중복 Reconcile 방지 |
| `queue`         | `RateLimitingQueue` — RS 키 기반 작업 큐, 지수 백오프 재시도                            |

* `rsLister`/`podLister`는 구조체 리터럴에 포함되지 않고, 아래 Informer 핸들러 등록 단계에서 별도 할당
* `expectations` 상세:
    * 문제: Pod 생성 요청 후 Informer 캐시 갱신 전에 Reconcile이 재트리거되면 중복 생성 위험
    * 생성(add): 단순 카운터 — Pod Add 이벤트마다 1 차감
    * 삭제(del): UID 목록 추적 — 삭제 요청한 특정 Pod의 UID만 매칭 (단순 카운터면 다른 Pod 크래시가 잘못 차감 가능)
    * TTL: 응답 없어도 만료 후 Reconcile 재개 → 무한 블로킹 방지
    * 실제 동작(ExpectCreations, ExpectDeletions, SatisfiedExpectations 호출 흐름)은 5절 manageReplicas에서 상세 설명
* `burstReplicas`와 `slowStartBatch` 관계:
    * `burstReplicas`: 한 번의 Reconcile에서의 최대 수 제한 (바깥 한계)
    * `slowStartBatch`: 그 안에서 1→2→4→8 배치로 점진적 생성 (안의 속도 조절) — 6절에서 상세 설명

**Informer 이벤트 핸들러 등록**

> `pkg/controller/replicaset/replica_set.go:L223~249`

```go
rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    func(obj interface{}) { rsc.addRS(logger, obj) },
    UpdateFunc: func(oldObj, newObj interface{}) { rsc.updateRS(logger, oldObj, newObj) },
    DeleteFunc: func(obj interface{}) { rsc.deleteRS(logger, obj) },
})
rsInformer.Informer().AddIndexers(cache.Indexers{
    controllerUIDIndex: func(obj interface{}) ([]string, error) {
        controllerRef := metav1.GetControllerOf(obj.(*apps.ReplicaSet))
        if controllerRef == nil { return []string{}, nil }
        return []string{string(controllerRef.UID)}, nil
    },
})
rsc.rsIndexer = rsInformer.Informer().GetIndexer()
rsc.rsLister = rsInformer.Lister()
rsc.rsListerSynced = rsInformer.Informer().HasSynced
```

> `pkg/controller/replicaset/replica_set.go:L251~268`

```go
podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    func(obj interface{}) { rsc.addPod(logger, obj) },
    UpdateFunc: func(oldObj, newObj interface{}) { rsc.updatePod(logger, oldObj, newObj) },
    DeleteFunc: func(obj interface{}) { rsc.deletePod(logger, obj) },
})
rsc.podLister = podInformer.Lister()
rsc.podListerSynced = podInformer.Informer().HasSynced
controller.AddPodControllerIndexer(podInformer.Informer())
rsc.podIndexer = podInformer.Informer().GetIndexer()
```

* `AddEventHandler`로 이벤트 발생 시 호출될 핸들러 메서드를 등록 — 이벤트가 발생하면 Informer가 해당 핸들러를 호출
* 각 핸들러(`addRS`, `addPod` 등) 내부에서 최종적으로 **RS 키**를 큐에 삽입 — RS/Pod 이벤트 모두 "어느 RS를 reconcile할지"로 변환됨 (상세는 아래 핸들러 분석 참조)
* `controllerUIDIndex` — RS의 ownerReference(Deployment) UID로 인덱싱. `getReplicaSetsWithSameController`에서 같은 Deployment 아래 형제 RS를 빠르게 조회 (5절 삭제 경로에서 상세)
* `AddPodControllerIndexer` — Pod의 namespace + ownerReference UID로 인덱싱. `FilterPodsByOwner`에서 RS가 소유한 Pod 및 고아 Pod를 효율적으로 조회 (5절 Pod 목록 수집 단계에서 상세)
* `rsLister`/`podLister` — 로컬 캐시 읽기 전용 뷰 (API 서버 호출 없이 메모리 즉시 조회)
* `rsListerSynced`/`podListerSynced` — `Run()`에서 캐시 동기화 완료 대기 시 사용 (4.4에서 상세)

**유틸 함수: `enqueueRS`**

> `pkg/controller/replicaset/replica_set.go:L367`

```go
func (rsc *ReplicaSetController) enqueueRS(rs *apps.ReplicaSet) {
    key, err := controller.KeyFunc(rs)
    if err != nil { ... }
    rsc.queue.Add(key)
}
```

* RS 오브젝트 → `namespace/name` 키 추출 → `queue.Add(key)` 래퍼
* 코드에서 `enqueueRS(rs)`와 `queue.Add(rsKey)` 혼용
    * 입력이 RS 오브젝트면 `enqueueRS`, 키를 이미 갖고 있으면 `queue.Add` 직접 호출
    * 결과는 동일

**RS Informer 핸들러**

**`addRS()`**

> `pkg/controller/replicaset/replica_set.go:L387`

```go
func (rsc *ReplicaSetController) addRS(logger klog.Logger, obj interface{}) {
    rs := obj.(*apps.ReplicaSet)
    rsc.enqueueRS(rs)
}
```

* RS 키를 queue에 삽입 — 단순

**`updateRS()`**

> `pkg/controller/replicaset/replica_set.go:L394`

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
        * 시나리오: Pod 생성 API 실패(쿼터 초과 등) → 더 이상 새 이벤트 없음 → RS가 큐에 재삽입되지 않아 멈춤 → Informer resync가 RS에 대해 Update 이벤트 강제 발생 → `updateRS`가 무조건 enqueue → Reconcile 재개
* UID 변경 감지 (같은 이름으로 RS 재생성된 경우)
    * Informer가 Delete + Add 대신 하나의 Update로 합칠 수 있는 알려진 한계 (소스 TODO 주석 확인)
    * `curRS.UID != oldRS.UID` → `deleteRS(oldRS)` 호출로 이전 RS의 stale expectations 강제 삭제
    * 이 처리 없으면 새 RS가 이전 RS의 미충족 expectations를 물려받아 영구 블로킹 위험
    * `oldRS`를 `cache.DeletedFinalStateUnknown`으로 감싸 전달 — `deleteRS`의 tombstone 처리 코드 재사용

**`deleteRS()`**

> `pkg/controller/replicaset/replica_set.go:L429`

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
rsc.consistencyStore.Clear(types.NamespacedName{Namespace: rs.Namespace, Name: rs.Name}, rs.UID)
rsc.expectations.DeleteExpectations(logger, key)
rsc.queue.Add(key)
```

* **tombstone (`DeletedFinalStateUnknown`) 2단계 type assertion** — k8s 컨트롤러 Delete 핸들러 표준 패턴
    * 1단계: `obj`를 `*apps.ReplicaSet`으로 직접 assertion → 정상 Delete 이벤트
    * 2단계: 실패 시 `cache.DeletedFinalStateUnknown`으로 assertion → tombstone에서 `.Obj` 추출
* **tombstone이란** — Watch 연결 끊김 후 Relist 시 DeltaFIFO가 합성하는 가짜 Delete 이벤트 래퍼
    * **언제 발생하는가**: Watch 스트림이 끊기면 Informer가 자동으로 List API로 전체 목록을 다시 받아옴(Relist). 이 시점에 DeltaFIFO가 "이전에 알고 있던 키인데 새 목록에 없다" → 삭제된 것으로 추론해 tombstone 합성
    * **왜 `Obj`가 stale할 수 있는가**: `Obj`는 "마지막으로 알려진 오브젝트". Watch가 끊긴 사이에 오브젝트가 수정된 후 삭제됐다면 `Obj`는 수정 전 상태를 담고 있음 → 타입 이름이 `DeletedFinal**StateUnknown**`인 이유
    * **왜 2단계 assertion이 필요한가**: 정상 Watch 경로는 `*apps.ReplicaSet`으로 직접 전달, Watch 끊김 후 Relist 경로는 `DeletedFinalStateUnknown{Key, Obj}`로 감싸서 전달 — 두 경로가 달라서 각각 처리
    * 정의: `staging/src/k8s.io/client-go/tools/cache/delta_fifo.go:L793~800`
* `consistencyStore.Clear(...)` — 삭제된 RS의 `WroteAt` 기록 제거
    * **왜 필요한가**: `consistencyStore`는 Pod 쓰기 시점의 ResourceVersion을 기록해두고, `syncReplicaSet` 진입 시 `EnsureReady`로 캐시가 따라잡았는지 검사. RS가 삭제되면 더 이상 Reconcile이 일어나지 않으므로 추적 데이터가 불필요 — 제거하지 않으면 메모리 누수
    * **UID 매칭**: `Clear(owner, ownerUID)`는 UID가 일치할 때만 삭제 — 같은 이름의 새 RS(다른 UID)가 생성되어도 새 RS의 기록은 보존
* `expectations.DeleteExpectations(key)` — 해당 RS의 expectations 전부 삭제
* `queue.Add(key)` — Reconcile 기회 부여 (이미 삭제됐으면 즉시 종료)
    * `syncReplicaSet` 진입 시 `rsLister.Get` → `IsNotFound`이면 `return nil` (`replica_set.go:L778~783`)
    * 이벤트 핸들러 누락·캐시 지연 시에도 reconcile 루프가 동일하게 정리 — Reconcile 루프 내 방어 로직

**Pod Informer 핸들러**

**`addPod()`**

> `pkg/controller/replicaset/replica_set.go:L463`

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

* **`DeletionTimestamp` 있는 경우** → `deletePod()` 위임
    * Add 이벤트인데 왜 `DeletionTimestamp`가 있을 수 있는가:
        * **컨트롤러 재시작**: Informer가 List로 전체 Pod 목록을 받아 캐시를 채울 때, 이미 삭제 진행 중인 Pod(Soft Delete 상태)도 포함 → 캐시에 새로 올라오므로 Add 이벤트 발생
        * **Soft Delete 특성**: `kubectl delete` 등으로 삭제하면 API 서버가 `DeletionTimestamp`만 설정(Soft Delete). kubelet이 graceful termination 완료 후 finalizer 제거 시 etcd에서 실제 삭제. Soft Delete → 실제 삭제 사이에 Informer Relist 등으로 Add 이벤트가 발생 가능
* **ControllerRef 있는 경우** (일반)
    * `expectations.CreationObserved(rsKey)` — add 카운터 즉시 차감
    * owner RS 키를 queue에 삽입
* **ControllerRef 없는 고아 Pod**
    * 일반적으로 RS가 Pod를 생성할 때 `ownerReference`를 설정하므로 처음부터 ControllerRef가 있음. 고아 Pod가 생기는 경우:
        * 사용자가 직접 Pod를 수동 생성 (`kubectl run ...`)
        * RS를 `--cascade=orphan`으로 삭제 → Pod는 남지만 owner RS가 사라짐
        * 컨트롤러 버그·레이스 컨디션으로 ownerReference 설정 실패
    * 셀렉터 매칭 RS 전부 enqueue — 입양 여부 판단은 `syncReplicaSet`에서 수행
        * 매칭 기준: 같은 네임스페이스의 RS를 조회하고, `rs.Spec.Selector`가 Pod 레이블을 포함하는 RS만 대상
        * 셀렉터 겹침으로 여러 RS가 매칭될 수 있음 — 각 RS의 `syncReplicaSet`에서 ownerReference 패치를 시도하고, 먼저 성공한 RS가 소유권 획득
    * expectations 차감 안 함 — 고아 Pod 생성을 기다리는 컨트롤러 없음

**`deletePod()`**

> `pkg/controller/replicaset/replica_set.go:L581`

```go
pod, ok := obj.(*v1.Pod)
if !ok {
    tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
    if !ok { ... }
    pod, ok = tombstone.Obj.(*v1.Pod)
    if !ok { ... }
}

controllerRef := metav1.GetControllerOf(pod)
if controllerRef == nil {
    return  // 고아 Pod — 삭제에 관심 있는 컨트롤러 없음
}
rs := rsc.resolveControllerRef(pod.Namespace, controllerRef)
if rs == nil {
    return
}
rsKey, err := controller.KeyFunc(rs)
if err != nil { ... }
rsc.expectations.DeletionObserved(logger, rsKey, controller.PodKey(pod))
rsc.queue.Add(rsKey)
```

* tombstone 2단계 assertion — `deleteRS`와 동일 패턴. `*v1.Pod` → `DeletedFinalStateUnknown` → `.Obj` 추출
* ControllerRef 없는 고아 Pod → 무시 — 삭제에 관심 있는 컨트롤러 없음
* `resolveControllerRef` — ControllerRef가 가리키는 RS를 캐시에서 조회. `nil`이면 해당 RS가 이미 삭제됐거나 다른 Kind → return
* `expectations.DeletionObserved(rsKey, podUID)` — del expectations에서 해당 Pod UID 제거
* owner RS 키를 queue에 삽입 → `syncReplicaSet`에서 status 갱신 및 부족분 재생성 판단

**`updatePod()`**

> `pkg/controller/replicaset/replica_set.go:L506`

```go
if curPod.ResourceVersion == oldPod.ResourceVersion { return }

labelChanged := !reflect.DeepEqual(curPod.Labels, oldPod.Labels)
if curPod.DeletionTimestamp != nil {
    rsc.deletePod(logger, curPod)
    if labelChanged { rsc.deletePod(logger, oldPod) }
    return
}
curControllerRef := metav1.GetControllerOf(curPod)
oldControllerRef := metav1.GetControllerOf(oldPod)
controllerRefChanged := !reflect.DeepEqual(curControllerRef, oldControllerRef)
if controllerRefChanged && oldControllerRef != nil {
    if rs := rsc.resolveControllerRef(oldPod.Namespace, oldControllerRef); rs != nil {
        rsc.enqueueRS(rs)
    }
}
if curControllerRef != nil {
    rs := rsc.resolveControllerRef(curPod.Namespace, curControllerRef)
    if rs == nil {
        return
    }
    rsc.enqueueRS(rs)
    if !podutil.IsPodReady(oldPod) && podutil.IsPodReady(curPod) && rs.Spec.MinReadySeconds > 0 {
        rsc.enqueueRSAfter(rs, time.Duration(rs.Spec.MinReadySeconds)*time.Second)
    }
    return
}
if labelChanged || controllerRefChanged {
    rss := rsc.getPodReplicaSets(curPod)
    for _, rs := range rss { rsc.enqueueRS(rs) }
}
```

* `ResourceVersion` 동일 → 무시 (Informer 주기적 재동기화에 의한 중복 이벤트)
* **`DeletionTimestamp` 설정됨** → `deletePod()` 위임
    * **왜 즉시 처리하는가**: Pod 삭제는 2단계 — ① API 서버가 `DeletionTimestamp` 설정(Soft Delete) → ② kubelet이 preStop 훅·SIGTERM·grace period(기본 30초) 후 실제 종료·Delete 이벤트. 실제 Delete 이벤트를 기다리면 grace period 동안 대체 Pod가 생성되지 않아 가용성 공백 발생. `DeletionTimestamp` 시점에 즉시 "삭제된 것"으로 처리해 대체 Pod를 곧바로 생성
    * **레이블 변경 시 old/cur 모두 `deletePod()` 호출**: 레이블이 바뀌면 소속 RS가 달라질 수 있음. `deletePod(curPod)`로 새 레이블 기준 RS에 통보, `deletePod(oldPod)`로 이전 레이블 기준 RS에도 통보 → 양쪽 RS 모두 expectations 차감 및 부족분 재계산 가능. `DeletionTimestamp`는 한 번 설정되면 해제 불가이므로 oldPod의 `DeletionTimestamp`를 별도로 확인할 필요 없음
* **ControllerRef 변경됨** → 이전 owner RS도 enqueue
    * 소유권이 RS-A → RS-B로 이전된 경우, RS-A는 소유 Pod 감소·RS-B는 소유 Pod 증가 — 양쪽 모두 Reconcile 필요
* **ControllerRef 있음** (일반) → owner RS enqueue
    * `resolveControllerRef` — ControllerRef의 UID로 Informer 캐시에서 RS 조회. `nil` 반환 시: RS가 이미 삭제되어 캐시에 없거나, ControllerRef의 Kind가 ReplicaSet이 아닌 경우 → return
    * **MinReadySeconds 지연 enqueue**: Pod가 NotReady→Ready로 전환되고 `rs.Spec.MinReadySeconds > 0`이면 즉시 enqueue + `enqueueRSAfter`로 해당 초 뒤에 한 번 더 enqueue
        * **즉시 enqueue 이유**: `status.readyReplicas`는 MinReadySeconds와 무관하게 Ready 여부만으로 계산 — 즉시 반영 필요
        * **지연 enqueue 이유**: `status.availableReplicas`는 Ready 후 MinReadySeconds 경과해야 인정. 해당 시간 경과 후 `syncReplicaSet`을 다시 실행해야 Available 조건 갱신 가능 — 이 enqueue가 없으면 상태 변화를 유발할 이벤트가 없어 `availableReplicas`가 갱신되지 않음
* **고아 Pod의 레이블/ControllerRef 변경** → 매칭 RS 전부 enqueue (입양 가능성 확인)
    * 여기에 도달했다는 것은 `curControllerRef == nil` — 현재 Pod에 owner RS가 없는 고아 상태

**syncHandler 와이어링**

> `pkg/controller/replicaset/replica_set.go:L269`

```go
rsc.syncHandler = rsc.syncReplicaSet

return rsc
```

* `processNextWorkItem`에서 `rsc.syncHandler` 호출 → 실제 구현 `syncReplicaSet`으로 연결
* `return rsc` — 완성된 `*ReplicaSetController` 반환 → `newReplicaSetController`(4.2)에서 `controllerLoop`으로 감싸 `Controller` 인터페이스로 반환 → `RunControllers`에서 고루틴 시작 → `rsc.Run(ctx, workers)` 호출 — 4.4로 이어짐

### 4.4 Run() — 워커 시작

4.2의 `newReplicaSetController`에서 `rsc.Run`을 `controllerLoop`으로 감싸 `Controller` 인터페이스(`Name()`, `Run(ctx)`)로 반환. 실행 시점 호출 체인:

1. `RunControllers` (`controllermanager.go:L655`) — `controllers` 슬라이스 순회, 각 `controller.Run(ctx)`를 별도 고루틴으로 실행
2. `controllerLoop.Run(ctx)` (`controller_utils.go:L65`) — 저장된 함수 포인터 호출
3. 함수 포인터 = `newReplicaSetController`에서 등록한 클로저: `rsc.Run(ctx, ConcurrentRSSyncs)` (`apps.go:L415~417`)

함수 포인터 → `controllerLoop` 구조체 → `Controller` 인터페이스 → `RunControllers` 고루틴 실행이라는 간접 호출 구조. 실제 진입점은 아래 `Run()` 함수.

> `pkg/controller/replicaset/replica_set.go:L275`

```go
func (rsc *ReplicaSetController) Run(ctx context.Context, workers int) {
    defer utilruntime.HandleCrash()

    rsc.eventBroadcaster.StartStructuredLogging(3)
    rsc.eventBroadcaster.StartRecordingToSink(...)
    defer rsc.eventBroadcaster.Shutdown()

    var wg sync.WaitGroup
    defer func() {
        rsc.queue.ShutDown()
        wg.Wait()
    }()

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

* `defer HandleCrash()` — 패닉 발생 시 로깅 후 복구, 프로세스 전체 종료 방지
* 이벤트 브로드캐스터 시작 + `defer Shutdown()` — 종료 시 이벤트 파이프라인 정리
* **종료 시 정리 흐름**: `<-ctx.Done()` → defer 실행 → `queue.ShutDown()` (큐 닫아 워커의 `Get()` 즉시 반환) → `wg.Wait()` (모든 워커 종료 대기) → `eventBroadcaster.Shutdown()`
* `WaitForNamedCacheSyncWithContext` — RS/Pod Informer 캐시가 모두 동기화될 때까지 블로킹
    * 캐시 미동기화 상태에서 Reconcile 시작 시 stale 데이터로 잘못된 Pod 수 계산 위험
    * `false` 반환 시 early return — 두 가지 경우: ① List/Watch 오류 등으로 캐시 동기화 실패, ② 동기화 대기 중 ctx 취소(셧다운 시그널). 워커 고루틴 시작 전이므로 defer의 `wg.Wait()`는 즉시 반환
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
    utilruntime.HandleError(fmt.Errorf("sync %q failed with %v", key, err))
    rsc.queue.AddRateLimited(key)      // 실패: 지수 백오프 후 재삽입
    return true
}
```

* `worker`: `processNextWorkItem()`을 큐 종료 시까지 무한 반복
    * 소스 주석: "syncHandler is never invoked concurrently with the same key" — `queue.Get()`이 같은 key를 `Done()` 전까지 다시 내보내지 않아 동일 RS에 대한 동시 Reconcile 방지
* `queue.Get()` — 항목이 없으면 블로킹, 있으면 즉시 반환. `ShutDown()` 호출 시 `quit = true` 반환 → `false` 리턴 → worker 루프 종료
* `rsc.syncHandler` = `rsc.syncReplicaSet` (4.3의 와이어링) → 다음 섹션으로 연결
* **WorkQueue 2레이어 구조** — `RateLimitingInterface`는 두 개의 독립된 상태 관리:

| 레이어          | 관련 메서드                               | 추적 내용                             |
|--------------|--------------------------------------|-----------------------------------|
| Base Queue   | `Get()`, `Done(key)`                 | 현재 처리 중인 키 (`processing` 집합)      |
| Rate Limiter | `AddRateLimited(key)`, `Forget(key)` | 키별 실패 횟수 (`failures map[key]int`) |

* `defer queue.Done(key)` — Base Queue 레이어: "이 key 처리 완료" 표시. `Done()` 호출 전까지 같은 key가 다시 `Get()`으로 나오지 않음
    * `Done`과 `Forget`은 완전히 다른 레이어 — `Forget`만 호출하고 `Done` 생략 시 해당 키가 `processing`에 남아 큐에서 재처리 불가
* 성공 시 `queue.Forget(key)` — Rate Limiter 레이어: `delete(failures, key)`. 큐에서 제거하는 게 아니라 실패 카운터만 삭제
    * `Forget` 없이 성공하면 `failures` 맵에 이전 실패 횟수가 남음 → 다음 실패 시 누적된 backoff 적용
    * `Forget` 호출 → 카운터 초기화 → 다음 실패부터 5ms로 재시작
* 실패 시 `HandleError` 로깅 + `queue.AddRateLimited(key)` — 기본 backoff: `baseDelay(5ms) * 2^실패횟수`, 최대 1000s

## 5. `syncReplicaSet()` — Reconcile 루프

* WorkQueue에서 꺼낸 키(namespace/name)로 실제 조정을 수행하는 핵심 함수

### 처리 흐름 개요

```
// pkg/controller/replicaset/replica_set.go:L755
syncReplicaSet(key)
  │
  ├─ consistencyStore.EnsureReady — 캐시가 이전 쓰기를 따라잡았는지 검증
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

> `pkg/controller/replicaset/replica_set.go:L767~787`

```go
if err := rsc.consistencyStore.EnsureReady(rsNamespacedName); err != nil {
    return err  // 캐시가 이전 쓰기를 아직 반영 못 함 → 재큐잉
}

rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name)
if apierrors.IsNotFound(err) {
    rsc.consistencyStore.Clear(rsNamespacedName, "")
    rsc.expectations.DeleteExpectations(logger, key)
    return nil
}
```

* `consistencyStore.EnsureReady(owner)` — 이전 Reconcile에서 API 서버에 쓴 ResourceVersion을 Informer 캐시가 따라잡았는지 검증
    * 미준비 시 `error` 반환 → worker가 `AddRateLimited`로 재큐잉 → 캐시 동기화 후 재처리
    * stale 캐시 기반으로 잘못된 diff 계산 방지 (예: 방금 생성한 Pod가 캐시에 아직 없어 중복 생성)
* 로컬 캐시에서 RS 조회 — API 서버 호출 없음
* `IsNotFound` → RS 이미 삭제됨 → `consistencyStore.Clear` + expectations 정리 후 정상 종료
    * `Clear(owner, "")` — UID 빈 문자열 = 무조건 삭제. `deleteRS`의 `Clear(owner, rs.UID)`(UID 매칭)와 다름
    * deleteRS 핸들러가 먼저 처리하면 이미 제거됨 → 여기서는 deleteRS 미호출 시 대비하는 방어 경로
    * `nil` 반환으로 worker가 `queue.Forget(key)` 호출 → 재시도 카운터 초기화 (4.5 참조)
* 조회한 RS는 "Informer가 마지막으로 수신한 스냅샷" — 실제 현재 상태와 약간의 차이 가능

#### 2단계 — Pod 목록 수집 & 분류

> `pkg/controller/replicaset/replica_set.go:L797~813`

```go
allRSPods, err := controller.FilterPodsByOwner(rsc.podIndexer, &rs.ObjectMeta, rsc.Kind, true)

allActivePods := controller.FilterActivePods(logger, allRSPods)
activePods, err := rsc.claimPods(ctx, rs, selector, allActivePods)

allTerminatingPods := controller.FilterTerminatingPods(allRSPods)
terminatingPods := controller.FilterClaimedPods(rs, selector, allTerminatingPods)
```

* `FilterPodsByOwner`: 이 RS 소유 Pod 전체 + 네임스페이스 내 고아 Pod 전체 수집
    * 고아 Pod = ownerRef가 없는 Pod — UID와 무관하게 네임스페이스 단위로 전부 반환
    * 셀렉터 매칭은 이후 `claimPods`에서 수행 — 여기서는 후보군만 수집
* `FilterActivePods`: `Succeeded` / `Failed` / `DeletionTimestamp` 설정된 Pod 제외 → Active Pod만 추출
    * Active = `Phase`가 `Running` / `Pending` / `Unknown`이고 삭제 요청 없는 Pod
    * "이미 삭제된 Pod"는 캐시에 존재하지 않음 — `DeletionTimestamp != nil`은 삭제 요청됐지만 kubelet이 아직 종료 처리 중인 Pod
* `claimPods`: 각 Pod의 ownerRef 상태에 따라 분류 → **API 서버 PATCH**로 입양/방출 수행 → 이 RS가 책임질 Pod만 반환
    * **이 RS 소유 Pod** (`ownerRef.UID == rs.UID`): 셀렉터 매칭 → 유지 / 셀렉터 불일치 + RS 미삭제 → ownerRef 제거 PATCH (방출) / 셀렉터 불일치 + RS 삭제 중 → 방출 건너뜀
    * **다른 컨트롤러 소유 Pod**: 무시
    * **고아 Pod** (ownerRef 없음): 셀렉터 매칭 + RS 미삭제 + Pod 미삭제 → ownerRef 추가 PATCH (입양) / 조건 불일치 → 무시
    * 입양 성공한 고아 Pod도 반환값에 포함 → 이후 diff 계산에 반영
    * 분류만 하는 것이 아니라 실제 etcd 쓰기 발생 — 2단계(Pod 목록 수집 & 분류)에서 이미 상태 변경 가능
* **Terminating Pod** — `DeletionTimestamp != nil` + Phase가 `Succeeded`/`Failed` 아닌 Pod (삭제 요청됐지만 kubelet이 아직 종료 처리 중)
    * `IsPodActive`가 `DeletionTimestamp != nil`을 항상 제외 → feature gate와 무관하게 `activePods`에 포함되지 않음 → diff 계산에서 빠지므로 대체 Pod 즉시 생성
    * **RS Reconcile 핵심 동작(diff 계산 → Pod 생성/삭제)에는 영향 없음** — `activePods`만 사용
    * feature gate `DeploymentReplicaSetTerminatingReplicas`: Terminating Pod를 별도 수집하여 `calculateStatus`에 전달 → RS status의 `TerminatingReplicas` 필드에 count 노출
        * RS 입장에서는 메타데이터 수집 및 노출이 전부
        * Deployment 컨트롤러 등 상위 컨트롤러가 이 필드를 읽어 "현재 몇 개가 종료 중"인지 파악 → 롤링 업데이트 중 과도한 Pod 생성 방지 등 정밀한 판단에 활용

#### 3단계 — expectations 확인 & manageReplicas 호출 조건

> `pkg/controller/replicaset/replica_set.go:L789, L818~820`

```go
rsNeedsSync := rsc.expectations.SatisfiedExpectations(logger, key)
...
if rsNeedsSync && rs.DeletionTimestamp == nil {
    manageReplicasErr = rsc.manageReplicas(ctx, activePods, rs)
}
```

* `rsNeedsSync = true` = "이전 요청이 캐시에 반영되어 diff 계산이 안전한 상태"
    * expectations은 "할 일이 있는가"가 아니라 **"이전에 요청한 Pod 생성/삭제가 Informer 캐시에 반영됐는가"**를 추적
    * 미충족 시 `manageReplicas`를 건너뛰는 이유: 캐시가 stale → 같은 diff 재계산 → Pod 중복 생성/삭제 위험
    * `Fulfilled()`: add/del expectations가 모두 0 이하 — 이전 요청이 캐시에 반영 완료 (음수 가능: Informer 이벤트가 expectations 등록보다 먼저 도착한 경우)
    * `isExpired()`: expectations TTL 만료 — 무한 블로킹 방지
    * expectations 자체가 없음 — 처음 생성된 RS 또는 이전 expectations 정리 완료
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
diff *= -1
if diff > rsc.burstReplicas {
    diff = rsc.burstReplicas
}
rsc.expectations.ExpectCreations(logger, rsKey, diff)
successfulCreations, err := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() error {
    err := rsc.podControl.CreatePods(...)
    if err != nil {
        if apierrors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
            return nil  // 네임스페이스 종료 중 → 성공 처리, 재시도 안 함
        }
    }
    return err
})
// slowStartBatch 중단으로 시도 못 한 Pod 수만큼 expectations 즉시 차감
if skippedPods := diff - successfulCreations; skippedPods > 0 {
    for i := 0; i < skippedPods; i++ {
        rsc.expectations.CreationObserved(logger, rsKey)
    }
}
```

* `burstReplicas` 상한 적용 (기본 500)
* `ExpectCreations(diff)` **먼저** 등록 → 이후 Pod Add 이벤트가 차감
* 배치 중단 시 건너뛴 수만큼 즉시 차감 → Informer 이벤트 없이 선제 차감하지 않으면 블로킹
    * `slowStartBatch`는 배치 중 에러 발생 시 즉시 반환 — 이후 배치는 시도 자체를 안 함                                                                 
    * `skippedPods = diff - successfulCreations` → 시도 못 한 수만큼 `CreationObserved` 호출 (정상 완료 시 `skippedPods = 0`이므로 진입 안 함)
* 네임스페이스 삭제 중 Pod 생성 실패(`NamespaceTerminatingCause`) → 콜백이 `nil` 반환으로 성공 처리
    * `slowStartBatch` 관점에서 에러 없음 → 배치 중단 없이 모든 `diff`개 시도 완료
    * `successfulCreations == diff` → `skippedPods = 0` → `CreationObserved` 미호출
    * 실제 Pod는 생성되지 않았으므로 Informer Add 이벤트 없음 → expectations 미충족 유지 → TTL 만료로 자연 해소
    * 의도: `nil` 반환 없이 에러를 그대로 전파하면 `skippedPods > 0` → `CreationObserved` 일부 호출 → 다음 resync 때 재시도 → 또 실패 → 무한 반복 + API 스팸

**삭제 경로 (diff > 0)**

> `pkg/controller/replicaset/replica_set.go:L700~747`

```go
if diff > rsc.burstReplicas {
    diff = rsc.burstReplicas
}
podsToDelete := getPodsToDelete(activePods, relatedPods, diff)
rsc.expectations.ExpectDeletions(logger, rsKey, getPodKeys(podsToDelete))
errCh := make(chan error, diff)
var wg sync.WaitGroup
wg.Add(diff)
for _, pod := range podsToDelete {
    go func(targetPod *v1.Pod) {
        defer wg.Done()
        if err := rsc.podControl.DeletePod(...); err != nil {
            podKey := controller.PodKey(targetPod)
            rsc.expectations.DeletionObserved(logger, rsKey, podKey)
            if !apierrors.IsNotFound(err) {
                errCh <- err
            }
        }
    }(pod)
}
wg.Wait()

select {
case err := <-errCh:
    if err != nil { return err }
default:
}
```

* `ExpectDeletions` **먼저** 등록 (UID 목록) → Pod Delete 이벤트가 차감
* `getPodsToDelete`: 우선순위 정렬 후 앞에서 `diff`개 선정 (`ActivePodsWithRanks.Less` 기준)
    1. 노드 미배정 Pod (`Spec.NodeName == ""`)
    2. Phase 순: Pending < Unknown < Running
    3. Not Ready < Ready
    4. `pod-deletion-cost` 어노테이션 낮은 Pod (feature gate `PodDeletionCost`)
    5. 같은 노드에 관련 Pod가 많이 몰린 Pod — `relatedPods`(같은 owner의 다른 RS Pod 포함)로 노드별 분포 계산, 분산 유지
    6. Ready 시간이 짧은 Pod (막 Ready된 Pod 먼저)
    7. 재시작 횟수가 많은 Pod
    8. 생성 시간이 짧은 Pod (최근에 만들어진 Pod 먼저)
* 삭제는 **병렬 goroutine** — 각 삭제 독립적, 실패해도 다른 삭제에 영향 없음
* 삭제 API 실패 시 `DeletionObserved` 즉시 호출 (에러 종류 무관)
    * `err != nil` = 이 호출로 인한 실제 삭제 없음 → Informer Delete 이벤트 없음 → 수동 차감 필요
* `IsNotFound` 이외의 오류 → `errCh` 전파 → 다음 Reconcile에서 재시도
* `IsNotFound` 오류 → `errCh` 전파 안 함 — Pod가 이미 삭제된 상태이므로 목표 달성, 재시도 불필요
* `errCh`: 실제 오류만 수집 → `wg.Wait()` 후 첫 번째 오류만 반환 — 오류가 대체로 동일(예: API 서버 불가)하므로 하나만 보고

**생성 vs 삭제 비대칭 설계**

|       | 생성                                           | 삭제                       |
|-------|----------------------------------------------|--------------------------|
| 실행 방식 | `slowStartBatch` — 배치 크기를 배로 늘리며 실행, 오류 시 중단 | 병렬 goroutine, 오류 시 개별 처리 |
| 이유    | 실패 원인(쿼터 부족 등)이 동일 → 전부 시도 전에 빠르게 중단이 유리     | 실패가 독립적 → 병렬 처리로 지연 없음   |

#### 5단계 — 상태 업데이트

> `pkg/controller/replicaset/replica_set.go:L821~841`

```go
rs = rs.DeepCopy()
now := rsc.clock.Now()
newStatus := calculateStatus(rs, activePods, terminatingPods, manageReplicasErr, rsc.controllerFeatures, now)
updatedRS, err := updateReplicaSetStatus(...)
if err != nil { return err }

rsc.consistencyStore.WroteAt(
    types.NamespacedName{Name: rs.Name, Namespace: rs.Namespace},
    rs.UID,
    replicaSetGroupResource,
    updatedRS.ResourceVersion,
)
if manageReplicasErr != nil { return manageReplicasErr }
```

* `manageReplicas` 오류 여부와 무관하게 **항상** 상태 업데이트 실행
* `rs.DeepCopy()` — Informer 캐시 원본 보호. status 필드 수정이 캐시를 오염시키지 않도록 복사본 사용
    * `updateReplicaSetStatus` 내부에서 `rs.Status = newStatus`로 객체를 직접 수정 → 캐시 포인터 그대로 넘기면 공유 캐시 오염
* `calculateStatus`: activePods/terminatingPods를 순회하며 status 필드 계산
    * `Replicas` = `len(activePods)`
    * `FullyLabeledReplicas` = activePods 중 `rs.Spec.Template.Labels`와 완전히 매칭되는 수
    * `ReadyReplicas` = activePods 중 `IsPodReady` 통과 수
    * `AvailableReplicas` = Ready Pod 중 `IsPodAvailable(pod, minReadySeconds, now)` 통과 수 — Ready 후 `MinReadySeconds`가 지난 Pod만 포함
    * `TerminatingReplicas` = `len(terminatingPods)` (feature gate `DeploymentReplicaSetTerminatingReplicas` + `EnableStatusTerminatingReplicas` 조건)
    * `ReplicaSetReplicaFailure` Condition: `manageReplicasErr != nil`이고 기존 Condition 없으면 설정 (`FailedCreate`/`FailedDelete`), `manageReplicasErr == nil`이고 기존 Condition 있으면 제거
* `updateReplicaSetStatus`: API 서버에 status 패치 (로컬 캐시가 아닌 실제 저장)
* `consistencyStore.WroteAt(...)` — status 패치로 변경된 ResourceVersion 기록 → 다음 Reconcile의 `EnsureReady`가 이 버전 이상을 요구
    * 1단계의 `EnsureReady`와 쌍을 이루는 메서드 — WroteAt이 기록한 ResourceVersion를 EnsureReady가 검증
* 오류 반환 순서 — 실행 순서(`manageReplicas` → `updateReplicaSetStatus`)와 반대
    * status 패치 오류 → `WroteAt` 호출 없이 즉시 반환 (먼저 반환)
        * status가 실제로 저장되지 않았으므로 ResourceVersion 변경 없음 → `WroteAt` 기록 의미 없음
        * `manageReplicasErr`보다 우선 반환 — 더 치명적, 이 상태로 재시도하면 정확한 상태 기반 불가
    * `manageReplicas` 오류 → status 패치 + `WroteAt` 완료 후 반환 (나중에 반환)
        * status는 "의도한 상태"가 아닌 **"현재 실제 상태"** 를 기록 — 생성/삭제 실패와 무관하게 지금 존재하는 Pod 수가 정확히 반영됨
        * `calculateStatus`에 `manageReplicasErr`를 넘겨 `ReplicaSetReplicaFailure` Condition도 함께 저장 → status 패치 내용 자체는 정확
        * 다음 Reconcile에서 "현재 Pod 수 + 이전 실패 여부"를 정확한 기반으로 재시도 가능

#### 6단계 — MinReadySeconds 재동기화 예약

> `pkg/controller/replicaset/replica_set.go:L822~824, L844~855`

```go
now := rsc.clock.Now()
newStatus := calculateStatus(rs, activePods, terminatingPods, manageReplicasErr, rsc.controllerFeatures, now)
// ...
if updatedRS.Spec.MinReadySeconds > 0 &&
    updatedRS.Status.ReadyReplicas != updatedRS.Status.AvailableReplicas {
    nextSyncDuration = ptr.To(time.Duration(updatedRS.Spec.MinReadySeconds) * time.Second)
    if nextCheck := controller.FindMinNextPodAvailabilityCheck(activePods, updatedRS.Spec.MinReadySeconds, now, rsc.clock); nextCheck != nil {
        nextSyncDuration = nextCheck
    }
}
if nextSyncDuration != nil {
    rsc.queue.AddAfter(key, *nextSyncDuration)
}
```

* `MinReadySeconds`: Pod가 Ready 상태로 N초 유지돼야 Available로 간주
    * 타임라인 (MinReadySeconds=30 기준):
        ```
        T=0s        Pod-A Ready 전환           → syncReplicaSet 실행 → ReadyReplicas=1, AvailableReplicas=0
        T=0~30s     Pod-A는 "Ready이지만 미 Available" 상태 (status 불일치)
        T=30s       Pod-A Available 조건 충족 (시간 경과만으로)
        T=30s 이후  syncReplicaSet이 다시 실행돼야 AvailableReplicas=1로 status 패치
                   ← 이 재실행을 보장하기 위한 예약이 6단계의 핵심
        ```
    * 30초 구간 동안 status 갱신이 필요하지만 Pod 이벤트 없음 → 별도 재동기화 예약 필요
* Ready지만 아직 Available 아닌 Pod 존재 → N초 후 재동기화 예약
* 예약 없으면 Available 전환 시 Informer 이벤트가 없어 status 업데이트 누락 가능
    * **주 경로** — 4.3의 `updatePod` 핸들러(L754~756): Pod NotReady→Ready 전환 시 `enqueueRSAfter(MinReadySeconds)`로 N초 후 재동기화 예약
    * **6단계(안전망)** — 주 경로가 실패할 수 있는 구조적 제약 대비:
        * **큐 키 경합**: RS에 여러 Pod이 있을 때 각 Pod의 `enqueueRSAfter`가 모두 **같은 RS 키**를 사용 → 타이머가 서로 덮어쓰거나 선점될 수 있음 (소스 주석 원문: `// we have one queue key for checking availability of all the pods`)
        * 컨트롤러 재시작으로 예약 타이머 유실
        * 이벤트 드롭·early sync (#39785)
    * `syncReplicaSet`이 매 reconcile 종료 시점에 보강 예약 → 이중 안전장치
    * 소스 주석: `// Plan the next availability check as a last line of defense against queue preemption ... or early sync` (L842~843) — 이벤트 드롭·큐 선점 상황 대비 안전장치
* `now` 변수(L823)를 `calculateStatus`(L824)와 `FindMinNextPodAvailabilityCheck`(L849)에 **동일하게 전달**
    * 의도: status 평가와 재동기화 타이밍 계산이 같은 기준 시각을 공유 → "status에서 미 Available로 집계한 Pod"와 "재동기화 예약이 필요한 Pod" 간 판정 일치
    * 시나리오 (MinReadySeconds=30, Pod-A ReadyTime=09:59:40):
        ```
        [올바른 동작 — now 공유]
        now = 10:00:00 (한 번 고정)
          calculateStatus: Pod-A 미 Available (09:59:40 + 30s = 10:00:10 > 10:00:00)
                        → AvailableReplicas에서 제외
          FindMin:        10:00:10 − 10:00:00 = 10s → 10초 후 재동기화 예약
          결과: 10:00:10경 reconcile → Pod-A Available 반영 → status 정확

        [잘못된 동작 — now 불공유 가정]
        calculateStatus(now=10:00:00): Pod-A 미 Available → AvailableReplicas 제외
        FindMin(now=10:00:11):         10:00:10 − 10:00:11 = −1s → nil 반환
                                      → fallback 30초 적용 → AddAfter at 10:00:11, duration=30s
        결과: 10:00:41경 reconcile — 실제 필요 시각(10:00:10)보다 약 30초 지연
             → 그 사이 availableReplicas status가 stale
        ```
    * ※ 실제 코드 경로상 `now` 두 읽기 간격은 μs 단위 — 위 11초 지연은 GC pause·극단 부하를 가정한 이론적 스트레스 시나리오. 공유 설계의 주 목적은 **판정 일관성 보증**이며, 실측 정밀도 향상은 부수 효과
    * 소스 주석: `// Use the same time for calculating status and nextSyncDuration.` (L822), `// Use the same point in time (now) ... to get matching availability for the pods.` (L848)
* `nextSyncDuration` 기본값: `MinReadySeconds * time.Second` — 최악 케이스 fallback
* `FindMinNextPodAvailabilityCheck` — 더 정밀한 재동기화 타이밍 계산 (controller_utils.go:L1078)
    * 1단계 `findMinNextPodAvailabilitySimpleCheck` — `lastOwnerStatusEvaluation`(= status 계산 시점 `now`) 기준으로 활성 Pod 중 가장 빨리 Available이 될 Pod와 남은 시간 선정
    * 2단계 — 선정된 Pod에 대해 `clock.Now()`(실제 현재 시각)로 재계산 → 1단계와 이번 호출 사이의 지연 보정
    * 2단계 결과가 nil (이미 Available 시점 지남) → `0` 반환 → 즉시 재동기화
    * 값을 반환하면 기본값 오버라이드 — Ready 전환 후 이미 경과한 시간을 반영하므로 기본값보다 짧거나 같음
    * 계산 예시 (MinReadySeconds=30, Pod-A ReadyTime=09:59:40, status 계산 now=10:00:00):
        ```
        1단계: findMinNextPodAvailabilitySimpleCheck(pods, 30s, lastOwnerStatusEvaluation=10:00:00)
          Pod-A: 09:59:40 + 30s − 10:00:00 = 10s (양수, 후보)
          → 선정: (Pod-A, 10s)

        2단계: nextPodAvailabilityCheck(Pod-A, 30s, clock.Now()=10:00:00.005)
          → 09:59:40 + 30s − 10:00:00.005 = 9.995s
          → 반환: 9.995s

        결과: queue.AddAfter(key, 9.995s) — fallback 30s 대신 정밀값 사용
        ```

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

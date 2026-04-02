# ReplicaSet Controller: Pod 생성과 조정 루프

## Deployment 적용부터 Pod 생성까지 — ReplicaSet Controller 내부 동작 분석

> * **대주제:** k8s 네트워크 스택 준비
> * **중주제:** ReplicaSet Controller

---

## 목차

1. 사전 학습 요약 — client-go Informer & WorkQueue
2. Deployment → ReplicaSet → Pod 계층 구조
3. ReplicaSet Controller 초기화 — Informer 연결
4. `syncReplicaSet()` — Reconcile 루프
5. `slowStartBatch()` — 왜 한꺼번에 생성하지 않는가
6. 고아 파드 입양 & 삭제 대상 선정
7. 마무리
8. 부록: 주요 코드 경로

---

## 1. 사전 학습 요약 — client-go Informer & WorkQueue

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

  Informer
    │  List (초기 전체 목록) + Watch (이후 변경 스트리밍)
    │  ◄── API Server
    ▼
 DeltaFIFO ──► Indexer (local cache)
    │
    ▼
 EventHandler (Add / Update / Delete)
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
| `tools/cache`    | `DeltaFIFO`             | 변경 이벤트 큐               |
| `util/workqueue` | `RateLimitingInterface` | 속도 제한이 있는 작업 큐         |

#### `SharedIndexInformer`

* Shared: 같은 오브젝트 타입에 대해 여러 컨트롤러가 하나의 Informer 공유
    * kube-controller-manager 안의 수십 개 컨트롤러가 Pod Informer 하나를 공유
* Index: 로컬 캐시(Indexer)에서 인덱스 기반 빠른 조회 지원
* List+Watch + 캐시 관리 + 인덱싱을 모두 담당

#### `DeltaFIFO`

* API 서버 이벤트를 EventHandler로 전달 전 쌓아두는 큐
* Delta: 이벤트 타입 5가지 — `Added`, `Updated`, `Deleted`, `Replaced`, `Sync`
* FIFO: 선입선출 순서 보장
* 같은 오브젝트의 이벤트가 여러 개 쌓이면 처리 전까지 병합 → 중복 처리 감소

#### `RateLimitingInterface`

* worker goroutine이 처리할 작업 키(`namespace/name`) 큐
* 중복 제거: 같은 키가 큐에 이미 있으면 추가 삽입 안 함
* 재시도 속도 제한: Reconcile 실패 시 지수 백오프로 재삽입
* 디커플링: 이벤트 수신 속도와 처리 속도 분리

---

## 2. Deployment → ReplicaSet → Pod 계층 구조

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

---

## 3. ReplicaSet Controller 초기화 — Informer 연결

* 진입점: `pkg/controller/replicaset/replica_set.go`
* `NewReplicaSetController()` — 컨트롤러 생성 및 Informer 이벤트 핸들러 등록

### `ReplicaSetController` 주요 필드

| 필드              | 타입                                              | 역할                                |
|-----------------|-------------------------------------------------|-----------------------------------|
| `kubeClient`    | `clientset.Interface`                           | API 서버 호출                         |
| `podControl`    | `controller.PodControlInterface`                | Pod 생성/삭제 추상화                     |
| `expectations`  | `*controller.UIDTrackingControllerExpectations` | 진행 중인 작업 추적 — 중복 reconcile 방지     |
| `queue`         | `workqueue.RateLimitingInterface`               | Reconcile 작업 큐                    |
| `burstReplicas` | `int`                                           | 한 번에 허용되는 최대 동시 Pod 작업 수 (기본 500) |
| `rsLister`      | `appslisters.ReplicaSetLister`                  | 로컬 캐시에서 ReplicaSet 조회             |
| `podLister`     | `corelisters.PodLister`                         | 로컬 캐시에서 Pod 조회                    |

#### `kubeClient`

* API 서버를 직접 호출하는 범용 클라이언트
* 로컬 캐시(`rsLister`, `podLister`)는 읽기 전용 → 생성/삭제 시 API 서버 직접 요청 필요

#### `podControl`

* `kubeClient`를 직접 쓰지 않고 Pod 생성/삭제를 이 인터페이스를 통해 수행
* 추상화 이유
    * 테스트 용이성: mock으로 교체 가능
    * 공통 처리: 이벤트 기록 등을 한 곳에서 처리

#### `expectations`

* 문제 상황: Pod 생성 요청 후 캐시 갱신 전에 Reconcile이 재트리거되면 중복 생성 위험
* 역할: "현재 N개 생성/삭제 요청 중"을 기록 → 미충족 시 `manageReplicas()` 스킵
* 기록 시점: `manageReplicas()`에서 Pod 생성/삭제 요청 시
* 해소 시점: Pod Informer EventHandler에서 해당 이벤트 수신 시 (큐 거치지 않고 즉시 처리)
* 생성(add): 단순 카운터 — Pod Add 이벤트마다 1 차감
* 삭제(del): UID 목록 추적 — 삭제 요청한 특정 Pod의 UID만 매칭
    * 단순 카운터일 경우, 다른 Pod 크래시가 삭제 expectations를 잘못 차감할 수 있음
* TTL: 응답 없어도 만료 후 Reconcile 재개 → 무한 블로킹 방지

#### `queue`

* 1절 `RateLimitingInterface`와 동일한 타입
* RS Informer와 Pod Informer 모두 이 큐에 **RS 키**를 삽입 — Reconcile 단위는 항상 RS
* RS Informer EventHandler 동작:
    * RS 키를 queue에 삽입 → 비동기 처리
* Pod Informer EventHandler 동작:
    * expectations 업데이트 → 즉시 처리
    * Pod의 owner RS 키를 queue에 삽입 → 비동기 처리

#### `burstReplicas`

* 한 번의 `manageReplicas()` 호출에서 처리 가능한 최대 Pod 수 (기본 500)
* `slowStartBatch`와의 관계:
    * `burstReplicas`: 한 번의 Reconcile에서의 최대 수 제한 (바깥 한계)
    * `slowStartBatch`: 그 안에서 1→2→4→8 배치로 점진적 생성 (안의 속도 조절) — 5절에서 상세 설명

#### `rsLister` / `podLister`

* API 서버가 아닌 로컬 캐시(Indexer)에서 읽는 읽기 전용 인터페이스
* 캐시를 쓰는 이유: 속도(메모리 즉시 조회), 부하 감소, 일관성(Informer가 Watch로 동기화)
* 쓰기(생성/삭제/수정)는 반드시 `kubeClient`/`podControl`로 API 서버 직접 요청

### Informer 이벤트 핸들러 등록

* `NewBaseController()` 에서 RS Informer와 Pod Informer 각각에 핸들러 등록
* 핸들러 함수는 모두 `ReplicaSetController`의 메서드 — 클로저로 캡처된 인스턴스(`rsc`)를 통해 호출

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

#### 유틸 함수: `enqueueRS`

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

#### RS Informer 핸들러

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
    * Informer가 Delete + Add 대신 하나의 Update로 합칠 수 있는 알려진 한계
    * `curRS.UID != oldRS.UID` → `deleteRS(oldRS)` 호출로 이전 RS의 stale expectations 강제 삭제
    * 이 처리 없으면 새 RS가 이전 RS의 미충족 expectations를 물려받아 영구 블로킹 위험

**`deleteRS()`**

> `pkg/controller/replicaset/replica_set.go:L452~459`

```go
rsc.expectations.DeleteExpectations(logger, key)
rsc.queue.Add(key)
```

* `expectations.DeleteExpectations(key)` — 해당 RS의 expectations 전부 삭제
* `queue.Add(key)` — Reconcile 기회 부여 (이미 삭제됐으면 즉시 종료)

#### Pod Informer 핸들러

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

---

## 4. `syncReplicaSet()` — Reconcile 루프

* WorkQueue에서 꺼낸 키(namespace/name)로 실제 조정을 수행하는 핵심 함수

### worker → syncReplicaSet 연결

```
NewBaseController()
  └─ rsc.syncHandler = rsc.syncReplicaSet     (L269)

Run(ctx, workers)
  ├─ WaitForNamedCacheSyncWithContext(...)      (L294) — Informer 캐시 동기화 대기
  └─ workers 수만큼 goroutine 생성             (L298~302)
       └─ worker(ctx)                           (L622~624)
            └─ for { processNextWorkItem(ctx) } (L623)
                 ├─ queue.Get()                 (L628)
                 ├─ syncHandler(ctx, key)       (L634) — == syncReplicaSet
                 ├─ 성공 → queue.Forget(key)    (L636)
                 └─ 실패 → queue.AddRateLimited(key) (L641) — 재시도 큐에 재삽입
```

> `pkg/controller/replicaset/replica_set.go:L269`

```go
rsc.syncHandler = rsc.syncReplicaSet
```

> `pkg/controller/replicaset/replica_set.go:L275~304`

```go
func (rsc *ReplicaSetController) Run(ctx context.Context, workers int) {
    ...
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

> `pkg/controller/replicaset/replica_set.go:L622~641`

```go
func (rsc *ReplicaSetController) worker(ctx context.Context) {
    for rsc.processNextWorkItem(ctx) {
    }
}

func (rsc *ReplicaSetController) processNextWorkItem(ctx context.Context) bool {
    key, quit := rsc.queue.Get()
    if quit { return false }
    defer rsc.queue.Done(key)

    err := rsc.syncHandler(ctx, key)
    if err == nil {
        rsc.queue.Forget(key)
        return true
    }
    rsc.queue.AddRateLimited(key)
    return true
}
```

* `syncHandler`를 필드로 저장하여 간접 호출 — 테스트 시 mock 함수로 교체 가능
* `Run()` 시작 시 Informer 캐시 동기화 완료를 대기한 후 worker 시작
* worker는 무한 루프로 큐에서 키를 꺼내 `syncHandler` 호출
* 성공 시 `queue.Forget(key)` — 재시도 카운터 초기화
* 실패 시 `queue.AddRateLimited(key)` — 지수 백오프 후 재시도

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

| 레이어 | 관련 메서드 | 추적 내용 |
|--------|------------|----------|
| Base Queue | `Get()`, `Done(key)` | 현재 처리 중인 키 (`processing` 집합) |
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

| | 생성 | 삭제 |
|---|---|---|
| 실행 방식 | `slowStartBatch` — 순차 배치, 오류 시 중단 | 병렬 goroutine, 오류 시 개별 처리 |
| 이유 | 실패 원인(쿼터 부족 등)이 동일 → 전부 시도 전에 빠르게 중단이 유리 | 실패가 독립적 → 병렬 처리로 지연 없음 |

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

## 5. `slowStartBatch()` — 왜 한꺼번에 생성하지 않는가

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

## 6. 고아 파드 입양 & 삭제 대상 선정

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

## 7. 마무리

### ReplicaSet Controller = k8s 컨트롤러 패턴의 교과서

* Informer → EventHandler → WorkQueue → Reconcile 패턴이 그대로 드러남
* 단일 책임: "현재 파드 수 = 원하는 파드 수" 만 유지
* expectations 로 이중 조정 방지, slowStartBatch 로 API 부하 분산

### 다음 세미나 예고 — 0006 kube-proxy iptables

* 같은 Informer + Reconcile 패턴이 **네트워크 레이어**에 적용됨
* kube-proxy는 Service / EndpointSlice 변경을 감지 → iptables 규칙 갱신
* "ClusterIP로 패킷을 보내면 어떻게 Pod에 도달하는가"

---

## 8. 부록: 주요 코드 경로

| 항목            | 경로                                                                         |
|---------------|----------------------------------------------------------------------------|
| 컨트롤러 메인       | `pkg/controller/replicaset/replica_set.go`                                 |
| 유틸리티          | `pkg/controller/replicaset/replica_set_utils.go`                           |
| Pod 생성/삭제 추상화 | `pkg/controller/controller_utils.go`                                       |
| expectations  | `pkg/controller/controller_utils.go` (`UIDTrackingControllerExpectations`) |
| WorkQueue     | `vendor/k8s.io/client-go/util/workqueue`                                   |

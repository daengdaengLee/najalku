# ReplicaSet Controller: Pod 생성과 조정 루프

## Deployment 적용부터 Pod 생성까지 — ReplicaSet Controller 내부 동작 분석

> **대주제:** k8s 네트워크 스택 준비
> **중주제:** ReplicaSet Controller

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

### 핵심 흐름

```
API Server
    │  List + Watch
    ▼
 DeltaFIFO ──► Indexer (local cache)
    │
    ▼
 EventHandler (Add / Update / Delete)
    │  key (namespace/name)
    ▼
 WorkQueue ──► worker goroutine ──► Reconcile()
```

### 핵심 타입 (`k8s.io/client-go`)

| 패키지              | 타입                      | 역할                     |
|------------------|-------------------------|------------------------|
| `tools/cache`    | `SharedIndexInformer`   | List+Watch → 로컬 캐시 동기화 |
| `tools/cache`    | `DeltaFIFO`             | 변경 이벤트 큐               |
| `util/workqueue` | `RateLimitingInterface` | 속도 제한이 있는 작업 큐         |

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

* 각 레이어가 분리된 이유
    * 롤링 업데이트: Deployment가 구 ReplicaSet ↔ 신 ReplicaSet 비율을 점진적으로 조정
    * 단일 책임: ReplicaSet은 "지정된 수의 동일 파드를 유지"만 담당

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

### Informer 이벤트 핸들러 등록

* (상세 코드 분석 예정)

---

## 4. `syncReplicaSet()` — Reconcile 루프

* WorkQueue에서 꺼낸 키(namespace/name)로 실제 조정을 수행하는 핵심 함수

### 처리 흐름 개요

```
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

* (상세 코드 분석 예정)

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

* 첫 배치 실패 시 나머지 배치 모두 중단
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

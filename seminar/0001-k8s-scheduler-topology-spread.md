# Pod는 왜 한 곳에 몰리는가
## Scheduler 기반 Pod 분산 전략 — 장애 사례로 배우는 TopologySpreadConstraints와 PDB

> **대주제:** k8s Control Plane 동작  
> **중주제:** Scheduler  
> **발표 시간:** 약 15~20분

---

## 목차

1. 오프닝: 실제 장애 사례
2. 장애 원인 분석: Scheduler는 무엇을 했는가
3. kube-scheduler의 Pod 배치 결정 흐름
4. 해결책 1 — TopologySpreadConstraints
5. 해결책 2 — PodDisruptionBudget
6. Karpenter와의 상호작용
7. 최종 권장 설정 정리
8. 마무리
9. 부록: 코드 분석
   - A. 스케줄러 메인 루프 → TopologySpread 플러그인 진입 경로
   - B. PodTopologySpread 플러그인 — 4단계 구조
   - C. PreFilter 단계 — 클러스터 현황 집계
   - D. Filter 단계 — skew 계산
   - E. Score 단계 — 분산 선호도 점수
   - F. 전체 코드 경로 요약

---

## 1. 오프닝: 실제 장애 사례

### 타임라인

```
[정상 운영 중]
  Node 1개에 Pod 3개가 모두 몰려있는 상태

        ↓

[Node TTL 만료]
  Karpenter가 해당 Node 강제 재기동

        ↓

[Pod 3개 동시 종료]
  클라이언트의 모든 API 요청 실패

        ↓ 약 6분 후

[새 Node 프로비저닝 + Pod 재시작 완료]
  서비스 정상화
```

### 핵심 질문

> **"Pod가 한 Node에 몰리지 않았다면, 장애를 막을 수 있었을까?"**

→ 답은 **Yes**. 그리고 그것이 오늘 발표의 핵심입니다.

---

## 2. 장애 원인 분석: Scheduler는 무엇을 했는가

### 왜 Pod가 한 곳에 몰렸는가

kube-scheduler는 기본적으로 Pod를 배치할 때 **가용 리소스**와 **기본 분산 정책**을 고려합니다.  
그러나 **명시적인 분산 제약 조건이 없으면**, 여러 이유로 특정 Node에 Pod가 집중될 수 있습니다.

- 초기 배포 시 Node 1개만 Ready 상태였던 경우
- 기존 Node에 리소스가 충분해서 신규 Node를 프로비저닝하지 않은 경우
- `podAntiAffinity` 등 분산 정책을 별도로 설정하지 않은 경우

### 두 가지 근본 원인

| 원인 | 설명 | 필요한 설정 |
|---|---|---|
| Pod 분산 미설정 | 애초에 Pod가 여러 Node에 나뉘지 않았음 | `topologySpreadConstraints` |
| 동시 종료 방어 미설정 | Node가 내려갈 때 Pod가 한꺼번에 종료됨 | `PodDisruptionBudget` |

---

## 3. kube-scheduler의 Pod 배치 결정 흐름

kube-scheduler는 새 Pod가 생성될 때 아래 두 단계를 거쳐 배치 Node를 결정합니다.

```
새 Pod 생성 요청
      ↓
[Filtering 단계]
  - 리소스 부족한 Node 제외
  - Taint/Toleration 불일치 Node 제외
  - topologySpreadConstraints 위반 Node 제외  ← 오늘의 핵심
      ↓
[Scoring 단계]
  - 남은 Node들에 점수 부여
  - 가장 높은 점수의 Node에 Pod 배치
      ↓
최종 Node 선택 → Pod 배치
```

`topologySpreadConstraints`는 `whenUnsatisfiable: DoNotSchedule`인 경우 **Filtering 단계**에서 평가됩니다.  
조건을 위반하는 Node는 후보에서 **아예 제외**됩니다.  
`whenUnsatisfiable: ScheduleAnyway`인 경우에는 **Scoring 단계**에서 선호도 점수로 반영됩니다.

---

## 4. 해결책 1 — TopologySpreadConstraints

### 개념

> "Pod를 특정 토폴로지(Node, AZ 등) 단위로 균등하게 분산시켜라"

### 핵심 파라미터

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-app
    minDomains: 3
```

| 파라미터 | 값 | 의미 |
|---|---|---|
| `maxSkew` | `1` | Node 간 Pod 수 차이를 최대 1개로 제한 |
| `topologyKey` | `kubernetes.io/hostname` | Node 단위로 분산 기준 설정 |
| `whenUnsatisfiable` | `DoNotSchedule` | 조건 불만족 시 스케줄링 거부 |
| `minDomains` | `3` | 가용 Node 수가 3개 미만이면 global minimum을 0으로 강제하여 분산을 강제 |

### skew 계산 방식

```
global minimum = 전체 Node 중 해당 App Pod 수가 가장 적은 Node의 Pod 수

skew(배치 시) = 해당 Node의 현재 Pod 수 + 1(신규 Pod) - global minimum
```

신규 Pod를 해당 Node에 배치했을 때의 상태를 시뮬레이션하기 때문에 현재 Pod 수에 +1을 더합니다.
`skew ≤ maxSkew`를 만족하는 Node만 배치 후보가 됩니다.

**예시 (Node 4개, Pod 7개, maxSkew: 1):**

```
현재 상태:
  Node A: 2개  Node B: 2개  Node C: 2개  Node D: 1개 ← global minimum(1)

신규 Pod 배치 시 skew:
  Node A: 2 + 1 - 1 = 2 > maxSkew(1) ❌
  Node B: 2 + 1 - 1 = 2 > maxSkew(1) ❌
  Node C: 2 + 1 - 1 = 2 > maxSkew(1) ❌
  Node D: 1 + 1 - 1 = 1 ≤ maxSkew(1) ✅

→ 신규 Pod는 Node D에만 배치 가능
```

### minDomains의 역할

`minDomains`는 **"가용 Node 수가 지정값 미만일 때 global minimum을 0으로 강제해 스케줄링을 제한"** 합니다.
이를 통해 Karpenter가 최소 `minDomains`개의 Node를 확보하도록 유도합니다.

**minDomains 없이 Node가 1개인 과도기 상황:**

```
Node A만 존재, Pod 3개 스케줄 필요

Pod 1 → Node A (1개) : global min = 0 → skew = 0 + 1 - 0 = 1 ✅
Pod 2 → Node A (2개) : global min = 1 → skew = 1 + 1 - 1 = 1 ✅
Pod 3 → Node A (3개) : global min = 2 → skew = 2 + 1 - 2 = 1 ✅

→ Pod 1, 2, 3 모두 Node A에 스케줄됨
→ 분산 없이 한 Node에 집중 (장애 위험)
```

**minDomains: 3 설정 후 동일 상황:**

```
실제 도메인 수(1) < minDomains(3)
→ global minimum을 0으로 강제

Pod 1 → Node A (1개) : skew = 0 + 1 - 0 = 1 ✅
Pod 2 → Node A (2개) : skew = 1 + 1 - 0 = 2 ❌ maxSkew(1) 초과!

→ Pod 2, 3이 Pending 상태로 대기
→ Karpenter가 Pending 감지 → 새 Node 프로비저닝
→ 실제 도메인 수 >= minDomains(3) → global minimum 정상화 → 균등 분산 적용
```

---

## 5. 해결책 2 — PodDisruptionBudget

### 개념

> "Node가 내려가더라도 최소 N개의 Pod는 반드시 유지하라"

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

### PDB의 동작 범위

PDB는 **Voluntary Disruption(자발적 중단)** 에만 적용됩니다.

| 구분 | 예시 | PDB 적용 |
|---|---|---|
| Voluntary (Eviction API 사용) | Node drain, Karpenter TTL evict | ✅ |
| Controller에 의한 Pod 교체 | rollout restart (Deployment 롤링 업데이트) | ❌ (Delete API 사용, rolling update strategy로 가용성 보장) |
| Involuntary | Node 하드웨어 장애, OOM kill, 커널 패닉 | ❌ |

### PDB 단독으로는 이번 장애를 막을 수 없었던 이유

```
Pod 3개가 모두 Node 1에 몰려있는 상태

Karpenter TTL → Node 1 drain 시도
      ↓
PDB: "minAvailable=2, 현재 3개 → 1개만 evict 허용"
Pod A evict → Node 1에 Pod B, C (2개) 남음
      ↓
PDB: "현재 2개 → 더 이상 evict 불가"
Node drain 블로킹 발생

      ↓ TTL 만료로 강제 종료 시

PDB 무시 → Node 1 즉시 종료
Pod B, C 동시 종료 → 서비스 중단 ❌
```

**결론:**  
Pod가 한 Node에 몰려있는 한, PDB는 drain을 늦출 수 있어도  
**Node 강제 종료 앞에서는 무력합니다.**

---

## 6. Karpenter와의 상호작용

### Karpenter의 스케일아웃 트리거

```
[HPA: Pod 증가 요청]
        ↓
[신규 Pod → Pending 상태 발생]
        ↓
[Karpenter가 Pending 감지 → 새 Node 프로비저닝]
        ↓
[Node Ready → Pod 스케줄링]
```

### topologySpreadConstraints + DoNotSchedule + Karpenter의 궁합

`whenUnsatisfiable: DoNotSchedule` 설정은 분산 조건을 만족하는 Node가 없을 때  
Pod를 **Pending** 상태로 만듭니다.  
이 Pending 상태가 Karpenter의 스케일아웃을 **자연스럽게 트리거**합니다.

```
기존 Node들이 skew 조건으로 인해 신규 Pod 수용 불가
        ↓
신규 Pod Pending
        ↓
Karpenter: 새 Node 추가
        ↓
새 Node에서 skew 조건 충족 → Pod 스케줄링
```

### rollout restart를 통한 수동 재분산

`topologySpreadConstraints`는 **신규 Pod 스케줄링 시점에만** 적용됩니다.  
기존 실행 중인 Pod는 자동으로 재배치되지 않습니다.

수동 재분산이 필요한 경우:

```bash
kubectl rollout restart deployment/my-app
```

섹션 7의 권장 설정(`maxUnavailable: 0`)으로 동작하므로 **무중단으로 재분산** 가능합니다.

```
초기: Node 1에 Pod A, B, C 모두 집중

Step 1: 새 Pod X 생성 (다른 Node에 배치) → Running 확인 후 Pod A 종료
Step 2: 새 Pod Y 생성 → Running 확인 후 Pod B 종료
Step 3: 새 Pod Z 생성 → Running 확인 후 Pod C 종료

결과: Pod X, Y, Z가 여러 Node에 균등 분산 ✅
```

---

## 7. 최종 권장 설정 정리

> **환경:** Node min 4개 (Karpenter), Pod min 3개 / max 30개

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0      # 배포 중에도 항상 min replicas 유지
  template:
    metadata:
      labels:
        app: my-app
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: my-app
          minDomains: 3      # min replicas와 동일하게 설정
      containers:
        - name: my-app
          image: my-app:latest
```

### PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2            # min replicas(3) - 1
  selector:
    matchLabels:
      app: my-app
```

### 설정별 역할 요약

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  topologySpreadConstraints                          │
│  └→ "애초에 Pod가 한 Node에 몰리지 않게"               │
│                                                     │
│  PodDisruptionBudget                                │
│  └→ "Node가 내려가도 서비스는 유지되게"                 │
│                                                     │
│  RollingUpdate (maxUnavailable: 0)                  │
│  └→ "배포 중에도 서비스는 유지되게"                     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 8. 마무리

### 이번 장애에 설정이 있었다면?

```
[초기 배포]
minDomains: 3 → 도메인 수(초기 1개) < 3 → global minimum = 0
→ Pod 1 스케줄 후 Pod 2부터 skew 초과로 Pending
→ Karpenter가 Pending 감지 → Node 추가 프로비저닝
→ 3개 Node 확보 후 Pod A, B, C가 Node 1, 2, 3에 각각 1개씩 분산 배치

        ↓ (이후 Node TTL 재기동 발생)

topologySpreadConstraints → Pod가 이미 Node 1, 2, 3에 1개씩 분산된 상태
        ↓
PDB → "minAvailable: 2 준수, 1개씩만 evict 허용"
        ↓
Node 1 종료 시 Pod A 1개만 영향, Node 2·3의 Pod B, C는 서비스 유지
        ↓
새 Node에 Pod A 재배치 완료

서비스 중단: 약 6분 → 0분 ✅
```

### 핵심 takeaway

| 설정 | 없었을 때 | 있었을 때 |
|---|---|---|
| `topologySpreadConstraints` | Pod 쏠림 발생 | Node 단위 균등 분산 보장 |
| `PDB` | Node 중단 시 다수 Pod 동시 종료 | 최소 가용 Pod 수 보장 |
| 두 설정 모두 | 이번 장애 발생 | 장애 없이 서비스 유지 |

> **분산은 "되면 좋은 것"이 아니라, 장애 예방의 필수 조건입니다.**

---

## 부록: 코드 분석

> 분석 기준 버전: `dd0958fece5c08e7438f4ce4f91678f1555a3b83`

---

### A. 스케줄러 메인 루프 → TopologySpread 플러그인 진입 경로

```
pkg/scheduler/schedule_one.go

ScheduleOne()           ← 스케줄링 루프에서 Pod 하나를 꺼내 처리
  └─ scheduleOnePod()
      └─ schedulingCycle()
          └─ schedulingAlgorithm()
              └─ sched.SchedulePod(...)   ← Scheduler 구조체의 함수 필드 (scheduler.go:87)
                     │                      기본값: sched.schedulePod (scheduler.go:127)
                     ▼
                 schedulePod()      ← Node 선택의 진짜 시작점
                  ├─ findNodesThatFitPod()   ← Filtering 단계
                  │   ├─ RunPreFilterPlugins()          ← PreFilter: 전체 클러스터 집계
                  │   └─ findNodesThatPassFilters()     ← Filter: 노드별 통과 판정
                  └─ prioritizeNodes()       ← Scoring 단계
                      ├─ RunPreScorePlugins()
                      └─ RunScorePlugins() + NormalizeScore()
```

**`schedulePod()`** (`schedule_one.go:568`)는 크게 두 단계로 나뉜다.
- `findNodesThatFitPod()` (`schedule_one.go:626`): Filter를 통과한 후보 노드 목록 생성
- `prioritizeNodes()` (`schedule_one.go:934`): 후보 노드들에 점수를 매겨 최적 노드 선택

Filter 단계에서 `findNodesThatPassFilters()` (`schedule_one.go:768`)는 모든 노드를 **병렬**로 검사한다.

```go
// schedule_one.go:806
status := schedFramework.RunFilterPluginsWithNominatedPods(ctx, state, pod, nodeInfo)
```

각 노드에 대해 등록된 모든 Filter 플러그인을 순서대로 실행하고, 하나라도 `Unschedulable`을 반환하면 그 노드는 탈락한다.

---

### B. PodTopologySpread 플러그인 — 4단계 구조

플러그인 파일: `pkg/scheduler/framework/plugins/podtopologyspread/`

| 단계 | 메서드 | 실행 시점 | 핵심 역할 |
|------|--------|-----------|-----------|
| PreFilter | `PreFilter()` | Pod 1개당 1회 | 클러스터 전체의 topology 현황 집계 → CycleState에 저장 |
| Filter | `Filter()` | 노드 1개당 1회 | skew 계산 → `maxSkew` 초과 노드 탈락 |
| PreScore | `PreScore()` | Pod 1개당 1회 | `ScheduleAnyway` constraints에 대한 topology 카운트 집계 (클러스터 전체 노드 기준) |
| Score + NormalizeScore | `Score()` / `NormalizeScore()` | 노드 1개당 1회 / 전체 1회 | pod 수가 적은 노드에 높은 점수 부여 |

---

### C. PreFilter 단계 — 클러스터 현황 집계

**`PreFilter()`** (`filtering.go:139`)

```go
func (pl *PodTopologySpread) PreFilter(...) (*fwk.PreFilterResult, *fwk.Status) {
    s, err := pl.calPreFilterState(ctx, pod, nodes)
    cycleState.Write(preFilterStateKey, s)  // 집계 결과를 CycleState에 저장
    return nil, nil
}
```

PreFilter의 핵심은 `calPreFilterState()` (`filtering.go:237`)다. 이 함수가 이후 모든 Filter 호출이 참조하는 **클러스터 스냅샷**을 만든다.

```go
// filtering.go:258-291 (핵심 요약)
processNode := func(n int) {
    // 1. 노드가 모든 topologyKey 라벨을 가지고 있는지 확인
    if !nodeLabelsMatchSpreadConstraints(node.Labels, constraints) {
        return
    }
    // 2. 각 constraint별로 이 노드의 topology value에 매칭되는 pod 수를 센다
    for i, c := range constraints {
        value := node.Labels[c.TopologyKey]
        count := countPodsMatchSelector(nodeInfo.GetPods(), c.Selector, pod.Namespace)
        tpCounts = append(tpCounts, topologyCount{topologyValue: value, constraintID: i, count: count})
    }
}
pl.parallelizer.Until(ctx, len(allNodes), processNode, pl.Name())  // 병렬 실행
```

집계 결과는 두 가지 자료구조에 담긴다.

#### `preFilterState` 자료구조 (`filtering.go:41`)

```go
type preFilterState struct {
    Constraints       []topologySpreadConstraint
    CriticalPaths     []*criticalPaths   // constraint별 최솟값 상위 2개
    TpValueToMatchNum []map[string]int   // topology value → 매칭 pod 수
}
```

- `TpValueToMatchNum`: `{"node-a": 2, "node-b": 1, "node-c": 0}` 같은 형태로, topology domain별 현재 pod 수 보관
- `CriticalPaths`: pod 수가 가장 적은 **상위 2개** domain만 추적 (크기 2 고정 배열)

#### `criticalPaths` 자료구조 (`filtering.go:97`)

```go
type criticalPaths [2]struct {
    TopologyValue string
    MatchNum      int
}
```

전체 topology를 저장하지 않고 **최솟값 2개만 유지**하는 이유: Filter 단계에서 필요한 것은 "global minimum"(= `paths[0].MatchNum`) 뿐이다. 2개를 유지하는 것은 선점(preemption) 계산 시 특정 노드의 Pod를 제거했을 때 minimum이 달라지는 경우를 처리하기 위해서다 (`filtering.go:90-95` 주석 참조).

---

### D. Filter 단계 — skew 계산

**`Filter()`** (`filtering.go:314`) — 노드 하나하나에 대해 호출된다.

```go
// filtering.go:329-355
for i, c := range s.Constraints {
    tpVal, ok := node.Labels[c.TopologyKey]
    if !ok {
        // topology key 라벨이 없는 노드 → 즉시 탈락
        return fwk.NewStatus(fwk.UnschedulableAndUnresolvable, ErrReasonNodeLabelNotMatch)
    }

    minMatchNum, _ := s.minMatchNum(i, c.MinDomains)  // global minimum 계산

    selfMatchNum := 0
    if c.Selector.Matches(podLabelSet) {
        selfMatchNum = 1  // 이 Pod 자신이 이 노드에 배치된다고 가정
    }

    matchNum := s.TpValueToMatchNum[i][tpVal]  // 이 노드 topology의 현재 pod 수
    skew := matchNum + selfMatchNum - minMatchNum

    if skew > int(c.MaxSkew) {
        return fwk.NewStatus(fwk.Unschedulable, ErrReasonConstraintsNotMatch)
    }
}
```

**skew 계산 공식:**

```
skew = (이 노드 topology의 현재 pod 수) + (이 Pod가 selector에 매칭되면 1, 아니면 0) - (global minimum)

통과 조건: skew ≤ maxSkew
```

`selfMatchNum`을 더하는 이유: Filter는 "이 Pod를 이 노드에 배치했을 때" 조건을 만족하는지 판단해야 한다. 아직 배치 전이므로 `+1`로 시뮬레이션한다.

#### `minMatchNum()` — minDomains 처리 (`filtering.go:55`)

```go
func (s *preFilterState) minMatchNum(constraintID int, minDomains int32) (int, error) {
    paths := s.CriticalPaths[constraintID]
    minMatchNum := paths[0].MatchNum           // 기본: pod 수가 가장 적은 domain의 값
    domainsNum := len(s.TpValueToMatchNum[constraintID])

    if domainsNum < int(minDomains) {
        minMatchNum = 0  // 가용 domain이 minDomains보다 적으면 global minimum을 0으로 강제
    }
    return minMatchNum, nil
}
```

`minMatchNum = 0`으로 강제하면 skew가 `matchNum + selfMatchNum`이 되어, 기존 도메인에 Pod를 추가할수록 skew가 빠르게 증가한다. 이는 분산 제약을 실질적으로 **강화**하여 기존 노드에 스케줄하기 어렵게 만들고, 결과적으로 Pod를 Pending 상태로 만들어 Karpenter가 새 Node를 프로비저닝하도록 유도한다. `minDomains: 3` 설정이 "가용 Node 3개 미만일 때 분산을 강제"처럼 동작하는 이유가 바로 이것이다.

---

### E. Score 단계 — 분산 선호도 점수

Score 단계는 `whenUnsatisfiable: ScheduleAnyway` constraints에 대해서만 작동한다. `DoNotSchedule` constraints는 Filter 단계에서만 평가되며 Score에는 참여하지 않는다. Filter를 통과한 노드들 사이에서 더 고르게 분산된 배치를 유도하는 역할을 한다.

#### PreScore — 집계 준비 (`scoring.go:118`)

`ScheduleAnyway` constraints를 수집하고, **클러스터 전체 노드**를 순회해 topology별 pod 수를 집계한다. `filteredNodes`(Filter 통과 노드)는 추적할 topology value(map의 키)를 결정하는 데만 사용되며, 실제 pod 수 집계(`processAllNode`)는 `allNodes` 전체를 대상으로 한다. 이와 함께 **TopologyNormalizingWeight**를 계산한다.

```go
// scoring.go:297
func topologyNormalizingWeight(size int) float64 {
    return math.Log(float64(size + 2))
}
```

이 가중치는 작은 topology(domain이 적은 것)의 점수가 큰 topology에 희석되지 않도록 보정한다. k8s가 지원하는 노드 수(최대 5000) 범위에서 결과는 `[1.09, 8.52]` 구간이다.

#### Score — 노드별 점수 (`scoring.go:199`)

```go
for i, c := range s.Constraints {
    if tpVal, ok := node.Labels[c.TopologyKey]; ok {
        var cnt int64
        if c.TopologyKey == v1.LabelHostname {
            // hostname 기준 분산: TopologyValueToPodCounts에 per-node 카운트가 없으므로
            // 이 노드의 Pod 목록에서 직접 계산
            cnt = int64(countPodsMatchSelector(nodeInfo.GetPods(), c.Selector, pod.Namespace))
        } else {
            cnt = *s.TopologyValueToPodCounts[i][tpVal]
        }
        score += scoreForCount(cnt, c.MaxSkew, s.TopologyNormalizingWeight[i])
        // scoreForCount = cnt * weight + (maxSkew - 1)
        // → pod 수(cnt)가 많을수록 높은 raw score
    }
}
```

#### NormalizeScore — 역전 정규화 (`scoring.go:229`)

```go
// scoring.go:265
scores[i].Score = fwk.MaxNodeScore * (maxScore + minScore - s) / maxScore
```

raw score가 **높을수록**(pod가 많을수록) 정규화 후 **낮은 최종 점수**를 받는다. pod가 적은 노드가 높은 점수를 받아 우선 선택된다.

```
raw score:   pod 많은 노드 → 높음  /  pod 적은 노드 → 낮음
                        ↓ NormalizeScore (역전)
final score: pod 많은 노드 → 낮음  /  pod 적은 노드 → 높음  ← 선택 우선
```

---

### F. 전체 코드 경로 요약

| 기능 | 파일 | 위치 |
|------|------|------|
| 스케줄링 루프 진입 | `pkg/scheduler/schedule_one.go` | `ScheduleOne():66` |
| Filter/Score 오케스트레이션 | `pkg/scheduler/schedule_one.go` | `schedulePod():568`, `findNodesThatFitPod():626`, `findNodesThatPassFilters():768` |
| PreFilter (클러스터 집계) | `pkg/scheduler/framework/plugins/podtopologyspread/filtering.go` | `PreFilter():139`, `calPreFilterState():237` |
| 핵심 자료구조 | `pkg/scheduler/framework/plugins/podtopologyspread/filtering.go` | `preFilterState:41`, `criticalPaths:97`, `minMatchNum():55` |
| Filter (skew 계산) | `pkg/scheduler/framework/plugins/podtopologyspread/filtering.go` | `Filter():314` |
| PreScore (점수 집계 준비) | `pkg/scheduler/framework/plugins/podtopologyspread/scoring.go` | `PreScore():118`, `initPreScoreState():61` |
| Score (노드 점수 계산) | `pkg/scheduler/framework/plugins/podtopologyspread/scoring.go` | `Score():199`, `topologyNormalizingWeight():297` |
| NormalizeScore (역전 정규화) | `pkg/scheduler/framework/plugins/podtopologyspread/scoring.go` | `NormalizeScore():229` |

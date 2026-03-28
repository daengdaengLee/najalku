# Descheduler : TopologySpreadConstraint 와 플러그인 비교

## 파드 재배치 전략 심화 — two-pointer 알고리즘으로 TSC 위반 해소 (Part B)

> **대주제:** k8s 클러스터 운영
> **중주제:** Descheduler

---

## 목차

1. 지난 시간 요약
2. RemovePodsViolatingTopologySpreadConstraint
3. 정리 및 비교
4. 마무리
5. 부록: 주요 코드 경로

---

## 1. 지난 시간 요약

* Part A에서 다룬 핵심 개념 리뷰

### Descheduler의 역할

* kube-scheduler 는 파드 배치를 일회성으로 결정
* Descheduler 는 이미 실행 중인 파드를 주기적으로 검사해, 현재 배치가 최적이 아니면 해당 파드를 **evict**
* evict된 파드는 kube-scheduler가 더 적합한 노드로 재배치

### Policy → Profile → Plugin 구조

```
Policy (설정 파일)
  └─ Profile (플러그인 조합에 이름을 붙인 단위)
       └─ Plugin (실제 eviction 판단 및 실행 로직)
```

### 세 가지 플러그인 인터페이스

| 인터페이스              | 질문                     | 구현체 예시             |
|--------------------|------------------------|--------------------|
| `DeschedulePlugin` | 이 파드를 evict해야 하는가?     | `RemoveFailedPods` |
| `BalancePlugin`    | 클러스터 전체를 어떻게 재분배할 것인가? | `RemoveDuplicates` |
| `EvictorPlugin`    | 이 파드를 evict해도 되는가?     | `DefaultEvictor`   |

### Handle → Evictor → DefaultEvictor 연결

* 플러그인은 `Handle`을 통해서만 클러스터에 접근
* `handle.Evictor()`가 돌려주는 `evictorImpl`은 초기화 시 `DefaultEvictor`의 Filter 함수들을 등록 -> 플러그인이 eviction을 요청할 때 자동으로
  DefaultEvictor의 판별 로직이 실행

```
DefaultEvictor (EvictorPlugin)
    ↓ 초기화 시 Filter / PreEvictionFilter 함수 등록
evictorImpl (Evictor)              ← handle.Evictor()가 리턴하는 실제 값
    ↓ Evict() 호출 시
PodEvictor                         ← 한도 체크 후 실제 Kubernetes Eviction API 호출
```

* evict 대상 판별은 두 단계로 나뉨
    * `Filter`(저비용, 후보 선정)로 먼저 좁힌 뒤
    * `PreEvictionFilter`(고비용, 최종 확인 — 대표적으로 NodeFit)를 통과한 파드만 실제 evict

### RemoveDuplicates

* `BalancePlugin` 구현체
* 같은 `ownerKey`(namespace + kind + name + imagesHash)를 가진 파드가 한 노드에 몰린 경우를 탐지하고,
  `upperAvg = ceil(전체 파드 수 / targetNode 수)` 기준으로 초과분을 즉시 evict

## 2. RemovePodsViolatingTopologySpreadConstraint

`RemovePodsViolatingTopologySpreadConstraint`는 `RemoveDuplicates`와 마찬가지로 `BalancePlugin`을 구현합니다. 클러스터 전체의 파드 분포를 보고 재분배가
필요한 파드를 evict합니다.

### 문제 정의

Kubernetes의 `TopologySpreadConstraint(TSC)`는 파드를 토폴로지 도메인(예: 가용 영역, 노드)에 고르게 분산시키는 규칙입니다. 이 규칙이 처음엔 만족되었더라도, 노드 추가/제거나
다른 eviction 이후 불균형이 생길 수 있습니다.

```
TopologySpreadConstraint:
  topologyKey: topology.kubernetes.io/zone
  maxSkew: 1

초기 상태 (균형):
  Zone A: [pod-1] [pod-2]   (2개)
  Zone B: [pod-3] [pod-4]   (2개)

→ Zone B의 노드가 다운되어 재스케줄링 발생

현재 상태 (위반):
  Zone A: [pod-1] [pod-2] [pod-3] [pod-4]   (4개)
  Zone B: (없음)                              (0개)
  skew = 4 - 0 = 4 > maxSkew(1) → TSC 위반
```

`RemovePodsViolatingTopologySpreadConstraint`는 이런 위반 상태를 감지하고 초과 도메인에서 파드를 evict해 재분배를 유도합니다.

### 핵심 용어 정리

| 용어                                   | 정의                                                                                    |
|--------------------------------------|---------------------------------------------------------------------------------------|
| **`TopologySpreadConstraint (TSC)`** | 파드 분산 규칙. `topologyKey`로 도메인을 나누고 `maxSkew`로 허용 불균형을 지정합니다.                           |
| **`topologyKey`**                    | 도메인 분류 기준이 되는 노드 레이블 키. 예: `"topology.kubernetes.io/zone"`.                           |
| **`maxSkew`**                        | 각 도메인의 파드 수와 전역 최솟값 사이의 허용 최대 차이. 예: maxSkew=1이면 어떤 도메인도 가장 적은 도메인보다 2개 이상 많을 수 없습니다. |
| **`topologyPair`**                   | `{key, value}` 쌍으로 하나의 토폴로지 도메인을 식별합니다. 예: `{key: "zone", value: "us-east-1a"}`.      |
| **`topology`**                       | `topologyPair` + 해당 도메인에 속한 파드 목록. `balanceDomains()`에서 도메인 단위로 다룹니다.                 |
| **`constraintTopologies`**           | 하나의 TSC에 대해 도메인별 파드 목록을 담는 맵. `topologyPair → []*v1.Pod`.                             |
| **`sortedDomains`**                  | `constraintTopologies`를 파드 수 기준 오름차순으로 정렬한 `[]topology`. two-pointer 알고리즘의 입력입니다.     |
| **`idealAvg`**                       | 이상적인 도메인당 평균 파드 수. `sumPods / len(constraintTopologies)`로 계산합니다.                      |
| **`podsForEviction`**                | 모든 TSC 처리 결과를 모은 최종 evict 대상 파드 집합. 중복 eviction 방지를 위해 맵으로 관리합니다.                     |

### 알고리즘 개요

```
Phase 1 — TSC별 도메인 집계 (네임스페이스 × TSC 순회)
  네임스페이스별로:
    해당 네임스페이스의 파드에서 고유 TSC 목록 수집
    TSC별로:
      constraintTopologies 초기화 (topologyKey를 가진 모든 노드 포함, 빈 도메인도 포함)
      각 파드를 알맞은 topologyPair에 할당해 파드 수 집계
      이미 균형이면 skip → balanceDomains() 호출

Phase 2 — two-pointer로 evict 대상 선정 (balanceDomains)
  idealAvg = sumPods / 도메인 수
  sortedDomains = 도메인을 파드 수 오름차순 정렬
  i=0(가장 작은 도메인), j=끝(가장 큰 도메인)
  while i < j:
    j 도메인이 idealAvg 이하면 j-- (줄 수 있는 게 없음)
    skew = sortedDomains[j].pods - sortedDomains[i].pods
    skew ≤ maxSkew면 i++ (이 쌍은 허용 범위 내)
    아니면 movePods 계산 후 j의 파드를 podsForEviction에 추가

Phase 3 — podsForEviction 실제 evict
```

### Balance() 코드 따라가기

> `pkg/framework/plugins/removepodsviolatingtopologyspreadconstraint/topologyspreadconstraint.go`

#### Phase 1: TSC별 도메인 집계

먼저 클러스터 전체 파드를 네임스페이스별로 묶습니다. (`L138-L147`)

```go
pods, err := podutil.ListPodsOnNodes(nodes, ...)  // L138: 전체 노드의 파드 수집
namespacedPods := podutil.GroupByNamespace(pods)   // L147: 네임스페이스별 그룹화
```

각 네임스페이스에서 고유한 TSC를 수집합니다. (`L154-L176`)

```go
for _, pod := range namespacedPods[namespace] {  // L155
    for _, constraint := range pod.Spec.TopologySpreadConstraints {
        if !allowedConstraints.Has(constraint.WhenUnsatisfiable) { continue }
        // 이미 수집한 것과 동일한 TSC면 skip (중복 제거)
        if hasIdenticalConstraints(...) { continue }  // L171
        namespaceTopologySpreadConstraints = append(..., tsc)
    }
}
```

TSC마다 `constraintTopologies`를 초기화할 때, **파드가 0개인 도메인도 미리 등록**합니다. (`L183-L192`)

```go
// L186-L192: 해당 topologyKey 레이블을 가진 모든 노드를 빈 슬라이스로 초기화
for _, node := range nodeMap {
    if val, ok := node.Labels[tsc.TopologyKey]; ok {
        constraintTopologies[topologyPair{key: tsc.TopologyKey, value: val}] = make([]*v1.Pod, 0)
    }
}
```

빈 도메인을 미리 넣는 이유: `idealAvg` 계산 시 파드가 없는 도메인도 분모에 포함해야 올바른 평균이 나오기 때문입니다.

이후 각 파드를 해당 topologyPair에 할당합니다. (`L197-L222`)

```go
for _, pod := range namespacedPods[namespace] {  // L197
    if !tsc.Selector.Matches(labels.Set(pod.Labels)) { continue }  // L203: 레이블 셀렉터 불일치 skip
    nodeValue := node.Labels[tsc.TopologyKey]
    topoPair := topologyPair{key: tsc.TopologyKey, value: nodeValue}  // L218
    constraintTopologies[topoPair] = append(..., pod)  // L220
    sumPods++
}
```

이미 균형이면 `balanceDomains()` 호출을 건너뜁니다. (`L223`)

```go
if topologyIsBalanced(constraintTopologies, tsc) { continue }  // L223
d.balanceDomains(ctx, podsForEviction, tsc, constraintTopologies, sumPods, nodes)  // L227
```

#### Phase 2: balanceDomains() — two-pointer

`balanceDomains()`는 `podsForEviction` 집합을 채우는 핵심 로직입니다. 실제 eviction은 모든 TSC 처리가 끝난 뒤 Phase 3에서 한꺼번에 수행합니다.

```go
func (d *...) balanceDomains(...) {  // L308
    idealAvg := sumPods / float64(len(constraintTopologies)) // L309
    sortedDomains := sortDomains(constraintTopologies, ...)   // L311: 파드 수 오름차순 정렬
    ...
    i := 0 // L320: 가장 작은 도메인 포인터
    j := len(sortedDomains) - 1 // L321: 가장 큰 도메인 포인터
```

`sortDomains()`는 도메인을 파드 수 오름차순으로 정렬하고, 각 도메인 안의 파드도 eviction 우선순위 순으로 정렬합니다. 슬라이스 **뒤쪽**이 우선 evict 대상이며, 뒤쪽부터 순서는 다음과
같습니다.

```
← 보호 (앞)                                         evict 우선 (뒤) →
  non-evictable | 셀렉터/affinity 있음 낮은→높은 우선순위 | 셀렉터/affinity 없음 낮은→높은 우선순위
```

즉, **셀렉터/affinity가 없고 우선순위가 높은 파드**가 가장 먼저 evict 대상이 됩니다.

> **참고**: 소스 코드의 주석(L419, L464)은 "낮은 우선순위를 먼저 evict"하겠다는 의도를 기술하고 있습니다. 그러나 실제 구현에서 `!comparePodsByPriority`는 낮은 우선순위를
> 슬라이스 앞쪽에, 높은 우선순위를 뒤쪽에 배치합니다. eviction이 뒤쪽부터 가져가므로 결과적으로 **높은 우선순위가 먼저 evict**됩니다. 코드 주석과 구현이 불일치하는 사례입니다.

two-pointer 루프: (`L322-L386`)

```go
for i < j {  // L322
    // j 도메인이 idealAvg 이하면 더 줄 수 없음 → j를 당김
    if float64(len(sortedDomains[j].pods)) <= idealAvg {  // L324
        j--; continue
    }

    skew := float64(len(sortedDomains[j].pods) - len(sortedDomains[i].pods)) // L329

    // skew가 maxSkew 이내면 이 쌍은 허용 범위 → i를 올림
    if int32(skew) <= tsc.MaxSkew {  // L332
        i++; continue
    }

    // 이동할 파드 수 계산 (세 가지 제약 중 최솟값)
    aboveAvg := math.Ceil(float64(len(sortedDomains[j].pods)) - idealAvg) // L343: j가 평균 초과분
    belowAvg := math.Ceil(idealAvg - float64(len(sortedDomains[i].pods))) // L344: i가 평균 미달분
    halfSkew := math.Ceil((skew - float64(tsc.MaxSkew)) / 2) // L346: skew 허용 초과분의 절반
    movePods := int(math.Min(math.Min(aboveAvg, belowAvg), halfSkew)) // L347

    if movePods <= 0 { i++; continue }  // L348
    // halfSkew: 한 쌍에서 과도하게 evict하는 것을 막는 제약.
    // i 도메인은 다른 iteration에서 j 역할의 도메인으로부터도 채워질 수 있으므로,
    // 현재 j에서 skew를 단독으로 0까지 줄이려 하면 전체적으로 과잉 eviction이 발생한다.

    // j의 뒤쪽 movePods개를 podsForEviction에 추가  (L362-L383)
    aboveToEvict := sortedDomains[j].pods[len(sortedDomains[j].pods)-movePods:]
    for k := range aboveToEvict {
        podsForEviction[aboveToEvict[k]] = struct{}{}  // L382
    }
    // 도메인 파드 수 갱신 (알고리즘 진행용 추적)
    sortedDomains[j].pods = sortedDomains[j].pods[:len(sortedDomains[j].pods)-movePods] // L384
    sortedDomains[i].pods = append(sortedDomains[i].pods, aboveToEvict...) // L385
}
```

예시 (`maxSkew=1`, 도메인 6개):

> **참고**: 파드 수가 바뀌어도 배열을 재정렬하지 않습니다. 괄호 안 숫자는 해당 인덱스의 현재 파드 수입니다.

```
초기 정렬: [2, 3, 5, 5, 7, 8]  idealAvg = 30/6 = 5

i=0(2), j=5(8): skew=6 > 1, aboveAvg=3, belowAvg=3, halfSkew=3 → movePods=3
  → idx 0: 2→5, idx 5: 8→5  [5, 3, 5, 5, 7, 5]

i=0(5), j=5(5): j ≤ idealAvg → j--
i=0(5), j=4(7): skew=2 > 1, aboveAvg=2, belowAvg=0 → movePods=0 → i++

i=1(3), j=4(7): skew=4 > 1, aboveAvg=2, belowAvg=2, halfSkew=2 → movePods=2
  → idx 1: 3→5, idx 4: 7→5  [5, 5, 5, 5, 5, 5]

i=1(5), j=4(5): j ≤ idealAvg → j-- ... i >= j → 종료
```

#### Phase 3: 실제 eviction

`podsForEviction`에 모인 파드를 순회하며 evict합니다. (`L231-L254`)

```go
for pod := range podsForEviction {  // L232
    if nodeLimitExceeded[pod.Spec.NodeName] { continue }  // L233: 노드 한도 초과 시 skip
    if !d.podFilter(pod) { continue }                     // L236: Filter 재확인

    if d.handle.Evictor().PreEvictionFilter(pod) {        // L240: 최종 확인 (NodeFit 등)
        err := d.handle.Evictor().Evict(ctx, pod, ...) // L241
        ...
    }
}
```

`RemoveDuplicates`와 달리, TSC 플러그인은 eviction을 두 단계로 분리합니다. `balanceDomains()`에서는 **후보만 선정**하고, 실제 eviction은 모든 TSC 처리가 끝난
뒤 한꺼번에 수행합니다. 이렇게 하면 여러 TSC가 동일한 파드를 두 번 evict하는 것을 방지할 수 있습니다.

## 3. 정리 및 비교

### 두 플러그인 비교

두 플러그인 모두 `BalancePlugin`을 구현하지만, 감지하는 문제와 해결 방식이 다릅니다.

#### 문제 정의

|            | RemoveDuplicates                | RemovePodsViolatingTopologySpreadConstraint |
|------------|---------------------------------|---------------------------------------------|
| **감지 대상**  | 같은 owner의 파드가 한 노드에 2개 이상 몰린 경우 | TSC의 maxSkew를 초과한 도메인 간 불균형                 |
| **범위**     | 노드 단위                           | 토폴로지 도메인 단위 (zone, region 등)                |
| **트리거 조건** | 명시적 규칙 없이 소유자 기준 자동 감지          | 파드에 TSC가 명시되어 있어야 동작                        |

#### 감지 방식

|             | RemoveDuplicates                          | RemovePodsViolatingTopologySpreadConstraint |
|-------------|-------------------------------------------|---------------------------------------------|
| **집계 단위**   | 노드별 `podContainerKeys` 비교                 | 네임스페이스별 TSC 수집 → 도메인별 파드 수 집계               |
| **빈 버킷 처리** | `duplicatePods`에 기록 없음 (Phase 2에서 처리)     | `constraintTopologies` 초기화 시 빈 도메인도 미리 등록   |
| **기준 계산**   | `upperAvg = ceil(전체 파드 수 / targetNode 수)` | `idealAvg = 전체 파드 수 / 도메인 수`                |

#### 알고리즘

|                 | RemoveDuplicates                     | RemovePodsViolatingTopologySpreadConstraint |
|-----------------|--------------------------------------|---------------------------------------------|
| **핵심 알고리즘**     | 단순 임계값 비교 (`len(pods)+1 > upperAvg`) | two-pointer on sorted domains               |
| **eviction 시점** | 초과분 발견 즉시 evict                      | 모든 TSC 처리 후 `podsForEviction` 한꺼번에 evict    |
| **즉시 evict 이유** | ownerKey별 독립 처리, 중복 eviction 위험 없음   | 여러 TSC가 동일 파드를 중복 선정할 수 있어 일괄 처리 필요         |

#### 시각적 비교

```
RemoveDuplicates
  노드 A: [web-1] [web-2] [web-3]   → web-3 evict (노드 내 초과)
  노드 B: [web-4]
  노드 C: (없음)                    → web-3 재배치 예상

RemovePodsViolatingTopologySpreadConstraint
  Zone A: [pod-1] [pod-2] [pod-3] [pod-4]   → pod-3, pod-4 evict 후보
  Zone B: (없음)                             → evict된 파드 재배치 예상
  (maxSkew=1 기준, two-pointer로 이동할 수 최소화)
```

### Descheduler 운영 시 주의사항

#### eviction은 재스케줄링을 보장하지 않습니다

Descheduler는 파드를 evict할 뿐, 어디로 재배치될지는 kube-scheduler가 결정합니다. evict 후 동일한 노드로 돌아올 수도 있습니다. 특히 `NodeFit` 옵션이 꺼져 있으면 갈 곳 없는
파드도 evict됩니다.

```
권장: DefaultEvictor의 NodeFit 옵션 활성화
→ PreEvictionFilter 단계에서 다른 노드에 스케줄 가능한지 확인 후 evict
```

#### PDB 설정 필수

PodDisruptionBudget(PDB)이 없으면 Descheduler가 동시에 너무 많은 파드를 evict해 서비스 중단이 발생할 수 있습니다. `PodEvictor`는 eviction 전 PDB를 확인하므로,
중요한 워크로드에는 반드시 PDB를 설정해야 합니다.

#### eviction 한도 설정

Policy에서 3가지 한도를 조합해 과도한 eviction을 방지합니다.

```yaml
maxNoOfPodsToEvictPerNode: 5        # 노드당 한 루프에 최대 eviction 수
maxNoOfPodsToEvictPerNamespace: 10  # 네임스페이스당 한 루프에 최대 eviction 수
maxNoOfPodsToEvictTotal: 50         # 전체 한 루프에 최대 eviction 수
```

#### 실행 전 dry-run 확인

Descheduler는 `--dry-run` 플래그를 지원합니다. 실제 eviction 없이 어떤 파드가 evict 대상이 되는지 로그로 확인할 수 있습니다. 운영 환경 적용 전에 반드시 dry-run을 먼저
실행해야 합니다.

#### Deployment 모드의 interval 설정

interval이 너무 짧으면 이전 eviction으로 인한 재스케줄링이 완료되기 전에 다음 루프가 돌아 불필요한 eviction이 반복될 수 있습니다. 클러스터 규모와 kube-scheduler의 처리 속도를
고려해 충분한 interval을 설정해야 합니다.

---

## 4. 마무리

### Descheduler가 없었다면?

```
kube-scheduler가 최초 배치한 직후 (균형):
  Node A: [web-1] [web-2]
  Node B: [web-3] [web-4]
  Node C: [web-5] [web-6]

        ↓ 시간이 지나며 노드 추가, 파드 eviction 반복

Descheduler 없이:
  Node A: [web-1] [web-2] [web-3] [web-4]   ← 쏠림 고착
  Node B: [web-5]
  Node C: [web-6]
  Node D: (빈 노드)

Descheduler 있을 때:
  → RemoveDuplicates가 Node A의 초과 파드를 evict
  → kube-scheduler가 Node D에 재배치
  Node A: [web-1] [web-2]
  Node B: [web-3] [web-5]
  Node C: [web-6]
  Node D: [web-4]   ← 균형 회복
```

### 핵심 takeaway

| 구성 요소                                         | 역할                                   | 핵심 메커니즘                        |
|-----------------------------------------------|--------------------------------------|--------------------------------|
| `RemoveDuplicates`                            | 같은 owner의 파드가 특정 노드에 몰린 경우 해소        | `upperAvg` 기준 초과분 즉시 evict     |
| `RemovePodsViolatingTopologySpreadConstraint` | TSC `maxSkew` 위반 상태 해소               | two-pointer로 최소 이동 evict 후보 선정 |
| `DefaultEvictor`                              | evict 부작용 방지                         | NodeFit · PDB 확인 후 최종 승인       |
| **Descheduler 전체**                            | kube-scheduler의 일회성 배치 결정을 주기적으로 재검토 | 매 루프마다 플러그인을 실행해 최적 상태 유지      |

> **kube-scheduler는 "지금 최선"을 고릅니다. Descheduler는 "지금도 최선인지"를 주기적으로 되묻습니다.**

---

## 부록: 주요 코드 경로

| 항목               | 경로                                                                                              |
|------------------|-------------------------------------------------------------------------------------------------|
| 진입점              | `cmd/descheduler/descheduler.go`                                                                |
| 서버 초기화           | `cmd/descheduler/app/server.go`                                                                 |
| 핵심 루프            | `pkg/descheduler/descheduler.go`                                                                |
| 플러그인 인터페이스       | `pkg/framework/types/types.go`                                                                  |
| Profile 초기화      | `pkg/framework/profile/profile.go`                                                              |
| DefaultEvictor   | `pkg/framework/plugins/defaultevictor/defaultevictor.go`                                        |
| Eviction 처리      | `pkg/descheduler/evictions/evictions.go`                                                        |
| RemoveDuplicates | `pkg/framework/plugins/removeduplicates/removeduplicates.go`                                    |
| TSC 플러그인         | `pkg/framework/plugins/removepodsviolatingtopologyspreadconstraint/topologyspreadconstraint.go` |

# Descheduler: Kubernetes 파드 재배치 전략
## kube-scheduler가 놓친 자리를 채우는 법 — 플러그인 아키텍처로 배우는 파드 재배치 전략

> **대주제:** k8s 클러스터 운영
> **중주제:** Descheduler
> **발표 시간:** 약 15~20분

---

## 목차

1. 왜 Descheduler가 필요한가?
2. 전체 동작 흐름
3. Plugin 시스템
4. [플러그인 1] RemoveDuplicates
5. [플러그인 2] RemovePodsViolatingTopologySpreadConstraint
6. 정리 및 비교
7. 마무리
- 부록: 주요 코드 경로

---

## 1. 왜 Descheduler가 필요한가?

kube-scheduler는 파드가 새로 생성될 때 노드를 선택한다. 이 결정은 **일회성**이다. 한번 배치된 파드는 노드가 더 적합한 상태로 바뀌어도 자동으로 이동하지 않는다.

다음 상황을 생각해보자.

```
초기 상태: 노드 3개, 각 노드에 파드가 고르게 배치됨

  Node A: ████████ (80% 사용)
  Node B: ████████ (80% 사용)
  Node C: ████████ (80% 사용)

→ 노드 D, E 추가 (빈 노드)

  Node A: ████████ (80%)
  Node B: ████████ (80%)
  Node C: ████████ (80%)
  Node D: (0%)          ← 새 파드만 배치됨
  Node E: (0%)          ← 새 파드만 배치됨
```

새 파드는 D, E에 배치되지만, A/B/C에 이미 있던 파드들은 그대로다. 시간이 지나면 같은 ReplicaSet 파드들이 특정 노드에 쏠리거나, 노드 간 부하 불균형이 고착된다.

kube-scheduler는 이 문제를 해결하지 않는다. **새 파드를 어디에 놓을지**만 결정할 뿐, **기존 파드를 옮기는 역할**은 없기 때문이다.

**Descheduler**는 이 간극을 채운다. 이미 실행 중인 파드를 주기적으로 검사해, 현재 배치가 최적이 아니라고 판단하면 해당 파드를 **evict(퇴출)** 한다. Evict된 파드는 kube-scheduler에 의해 다시 스케줄링되고, 이번엔 더 적합한 노드에 배치된다.

> **중요**: Descheduler는 파드를 직접 이동시키지 않는다. evict 후 재스케줄링은 kube-scheduler가 담당한다.

## 2. 전체 동작 흐름

#### 핵심 용어 정리

코드를 따라가기 전에 세 가지 용어의 관계를 먼저 잡고 가자.

```
Policy (설정 파일)
  └─ Profile (플러그인 조합의 단위, 여러 개 가능)
       └─ Plugin (실제 로직 단위, 여러 개 가능)
```

| 용어 | 역할 | 예시 |
|---|---|---|
| **Policy** | Descheduler 전체 설정. 어떤 Profile을 쓸지, 전역 eviction 한도는 얼마인지 정의 | `policy.yaml` |
| **Profile** | 함께 동작할 플러그인들의 묶음. 이름을 가지며, 하나의 Policy에 여러 Profile 공존 가능 | `"default"`, `"aggressive"` |
| **Plugin** | 실제 파드 eviction 판단 및 실행 로직. DeschedulePlugin / BalancePlugin / EvictorPlugin 중 하나를 구현 | `RemoveDuplicates`, `DefaultEvictor` |

Policy는 설정의 최상위 단위고, Profile은 플러그인 조합에 이름을 붙인 묶음이다. 동일한 플러그인이더라도 서로 다른 Profile에서 다른 설정으로 사용할 수 있다.

#### 호출 스택

```
main()                                            # cmd/descheduler/descheduler.go
└─ app.NewDeschedulerCommand()                    # cmd/descheduler/app/server.go
   ├─ PreRunE: SetupPlugins()                     # 플러그인 레지스트리 등록
   └─ RunE: app.Run()
      └─ descheduler.Run()                        # pkg/descheduler/descheduler.go
         ├─ LoadPolicyConfig()                    # Policy 설정 파일 파싱
         └─ RunDeschedulerStrategies()
            ├─ SharedInformerFactory 생성
            ├─ bootstrapDescheduler()
            │  ├─ NewPodEvictor()                 # eviction 실행기 생성
            │  ├─ Policy의 각 Profile마다 NewProfile() 호출
            │  │   ├─ 플러그인 인스턴스 생성 + Handle 주입
            │  │   ├─ 타입별로 분류: deschedulePlugins, balancePlugins, ...
            │  │   └─ profileRunner{RunDeschedulePlugins, RunBalancePlugins} 등록
            │  └─ sharedInformerFactory.Start()
            │     WaitForCacheSync()              # 캐시 동기화 완료까지 대기
            └─ wait.NonSlidingUntil(runLoop, interval, stopCh)
               └─ runDeschedulerLoop()
                  └─ runProfiles()               ← 아래 참조
```

#### Profile과 Plugin의 실행 흐름

`runProfiles()`는 각 Profile의 `RunDeschedulePlugins`, `RunBalancePlugins`를 순서대로 호출한다.

```
runProfiles(nodes)
  │
  ├─ [1단계] 모든 Profile의 Deschedule 플러그인 먼저 실행
  │    for each profileRunner:
  │      profileRunner.descheduleEPs(ctx, nodes)
  │        └─ profileImpl.RunDeschedulePlugins()
  │             └─ for each pl in deschedulePlugins:
  │                  pl.Deschedule(ctx, nodes)   ← 규칙 위반 파드 evict
  │
  └─ [2단계] 모든 Profile의 Balance 플러그인 나중에 실행
       for each profileRunner:
         profileRunner.balanceEPs(ctx, nodes)
           └─ profileImpl.RunBalancePlugins()
                └─ for each pl in balancePlugins:
                     pl.Balance(ctx, nodes)      ← 클러스터 재분배
```

Deschedule을 먼저 실행해 "명백한 문제(규칙 위반)" 파드를 제거하고, 그 후 Balance가 남은 파드들을 재분배한다. 두 단계를 분리함으로써 Balance가 이미 정리된 상태를 기준으로 균형을 계산할 수 있다.

#### 실행 모드

Descheduler는 세 가지 방식으로 배포할 수 있다.

| 배포 방식 | `DeschedulingInterval` | 동작 |
|---|---|---|
| **Job** | `0` | 한 번 실행 후 종료. 외부에서 직접 실행 |
| **CronJob** | `0` | 한 번 실행 후 종료. Kubernetes CronJob이 주기적으로 다시 실행 |
| **Deployment** | `> 0` | 프로세스가 계속 살아있으며 interval마다 반복 실행 |

CronJob과 Deployment는 주기적으로 실행한다는 점은 같지만, 이전 실행이 끝나지 않은 경우의 동작이 다르다.

| | 이전 실행이 interval 내에 끝난 경우 | 이전 실행이 interval을 초과한 경우 |
|---|---|---|
| **CronJob** (`concurrencyPolicy: Allow`, 기본값) | 정상 실행 | 중복 실행 |
| **CronJob** (`concurrencyPolicy: Forbid`) | 정상 실행 | 해당 주기 skip |
| **Deployment** | 남은 시간 대기 후 실행 | 끝나는 즉시 바로 실행 |

Deployment 방식은 단일 프로세스가 루프를 돌기 때문에 애초에 중복 실행이 불가능하다. `wait.NonSlidingUntil`은 이전 실행이 반드시 끝난 후 남은 interval 시간을 기다리고, 초과했다면 끝나는 즉시 다음 실행을 시작한다.

interval이 `0`이면 한 번 실행 후 context를 cancel해서 루프를 종료한다 (`pkg/descheduler/descheduler.go:546-548`).

#### SharedInformerFactory — 왜 캐시를 쓰는가?

플러그인들은 매 루프마다 클러스터의 노드/파드 목록을 조회한다. 매번 API 서버에 직접 요청하면 부하가 크기 때문에, client-go의 **Informer**를 사용해 로컬 캐시에서 읽는다.

```
API 서버 --Watch--> Informer (로컬 캐시) --List--> 플러그인
```

`WaitForCacheSync()`는 첫 루프가 시작되기 전, 캐시가 완전히 채워질 때까지 블로킹한다.

다음 섹션에서는 플러그인이 어떤 인터페이스로 구성되어 있는지, eviction 판단이 어떤 구조로 이루어지는지 살펴본다.

## 3. Plugin 시스템

호출 스택에서 `SetupPlugins()`이 플러그인을 레지스트리에 등록하고, `NewProfile()`이 각 플러그인을 인스턴스화하는 것을 봤다. **Profile**은 플러그인 조합에 이름을 붙인 단위로, 하나의 Policy에 여러 Profile을 정의할 수 있고 각 Profile은 독립적인 플러그인 집합을 가진다.

```
DeschedulerPolicy
  └─ Profile "default"
       ├─ DeschedulePlugin: RemoveFailedPods, ...
       ├─ BalancePlugin:    RemoveDuplicates, RemovePodsViolatingTopologySpreadConstraint, ...
       └─ EvictorPlugin:    DefaultEvictor
```

#### 플러그인 인터페이스

모든 플러그인은 `pkg/framework/types/types.go`에 정의된 인터페이스를 구현한다.

```go
// 모든 플러그인의 기반
type Plugin interface {
    Name() string
}

// 파드를 개별적으로 판단해서 evict — 규칙 위반 제거 목적
type DeschedulePlugin interface {
    Plugin
    Deschedule(ctx context.Context, nodes []*v1.Node) *Status
}

// 파드 전체를 보고 클러스터 균형을 맞춤 — 재분배 목적
type BalancePlugin interface {
    Plugin
    Balance(ctx context.Context, nodes []*v1.Node) *Status
}

// evict 가능한 파드인지 판별하는 기준을 제공 — DefaultEvictor가 구현
type EvictorPlugin interface {
    Plugin
    Filter(pod *v1.Pod) bool
    PreEvictionFilter(pod *v1.Pod) bool
}
```

세 인터페이스의 역할은 명확히 구분된다.

| 인터페이스 | 질문 | 구현체 예시 |
|---|---|---|
| `DeschedulePlugin` | 이 파드를 evict해야 하는가? | `RemoveFailedPods` |
| `BalancePlugin` | 클러스터 전체를 어떻게 재분배할 것인가? | `RemoveDuplicates` |
| `EvictorPlugin` | 이 파드를 evict해도 되는가? | `DefaultEvictor` |

#### Handle — 플러그인의 외부 접근 통로

플러그인은 생성 시 `Handle`을 주입받는다. 클러스터에 접근하는 **모든 수단**이 Handle을 통해서만 제공된다.

```go
type Handle interface {
    ClientSet() clientset.Interface           // Kubernetes API 직접 호출
    Evictor() Evictor                         // eviction 요청
    GetPodsAssignedToNodeFunc() ...           // 노드별 파드 목록 조회 (캐시)
    SharedInformerFactory() informers.SharedInformerFactory // 인포머 캐시
    MetricsCollector() *MetricsCollector      // CPU/메모리 사용량 수집
    // 외 PrometheusClient(), PluginInstanceID() 등
}
```

`Handle.Evictor()`의 리턴 타입은 `Evictor` 인터페이스다. `EvictorPlugin`과는 별개로, `Filter`/`PreEvictionFilter`에 더해 실제 eviction을 요청하는 `Evict()`까지 포함한다.

```go
type Evictor interface {
    Filter(*v1.Pod) bool
    PreEvictionFilter(*v1.Pod) bool
    Evict(context.Context, *v1.Pod, EvictOptions) error  // EvictorPlugin에는 없음
}
```

런타임에 `Handle.Evictor()`가 돌려주는 실제 값은 `evictorImpl`(`pkg/framework/profile/profile.go`)이다. `NewProfile()` 초기화 시 `EvictorPlugin`(DefaultEvictor)의 Filter 함수들을 `evictorImpl`에 미리 등록해둔다.

```
NewProfile() 초기화 시:
  DefaultEvictor.Filter            → evictorImpl.filter 에 등록
  DefaultEvictor.PreEvictionFilter → evictorImpl.preEvictionFilter 에 등록
```

이 덕분에 플러그인이 `handle.Evictor().Filter(pod)`를 호출하면 자동으로 DefaultEvictor의 판별 로직이 실행된다.

전체 연결 구조는 다음과 같다.

```
DefaultEvictor (EvictorPlugin)
    ↓ 초기화 시 Filter / PreEvictionFilter 함수 등록
evictorImpl (Evictor)              ← handle.Evictor()가 리턴하는 실제 값
    ↓ Evict() 호출 시
PodEvictor                         ← 한도 체크 후 실제 Kubernetes Eviction API 호출
```

#### DefaultEvictor — "누가 evict 대상인가"

`DefaultEvictor`는 `EvictorPlugin`의 구현체다. Deschedule/Balance 플러그인이 "이 파드를 evict하고 싶다"고 요청하면, DefaultEvictor가 "evict해도 되는 파드인가"를 판단한다.

```
RemoveDuplicates (BalancePlugin)
    │
    ├─ handle.Evictor().Filter(pod)          # 후보 선정 단계에서 반복 호출
    │       └─ evictorImpl.filter(pod)
    │            └─ DefaultEvictor.Filter()  # 초기화 때 등록된 함수 실행
    │
    └─ handle.Evictor().Evict(pod)           # evict 결정 후 호출
            ├─ evictorImpl.preEvictionFilter(pod)
            │    └─ DefaultEvictor.PreEvictionFilter()
            └─ 통과 시 PodEvictor → Kubernetes Eviction API
```

**Filter**: 아래 조건 중 하나라도 해당하면 evict 대상에서 제외한다.

| 항목 | 기본값 |
|---|---|
| Mirror 파드 / Static 파드 | 항상 제외 |
| Terminating 중인 파드 | 항상 제외 |
| system-critical 우선순위 파드 | 기본 제외 |
| DaemonSet 파드 | 기본 제외 |
| 로컬 스토리지(emptyDir 등) 사용 파드 | 기본 제외 |
| OwnerRef 없는 파드 (bare pod) | 기본 제외 |

단, 파드에 `descheduler.alpha.kubernetes.io/evict` 어노테이션이 있으면 위 규칙을 전부 무시하고 evict 대상으로 취급한다.

**PreEvictionFilter**: 실제 eviction 직전에 수행하는 최종 확인이다. `Filter`로 후보를 먼저 좁힌 뒤, 추려진 파드에만 실행하기 때문에 비용이 큰 검사를 여기에 둔다. 대표적으로 **NodeFit** 옵션이 여기서 동작한다. 해당 파드가 다른 노드에 스케줄될 수 있는지 확인하고, 갈 곳이 없으면 evict하지 않는다.

```
전체 파드
  └─ Filter (저비용, 많이 호출)         → 후보만 남김
       └─ PreEvictionFilter (고비용)    → 최종 확정된 파드에만 실행
            └─ Kubernetes Eviction API
```

## 4. [플러그인 1] RemoveDuplicates

### 문제 정의

같은 ReplicaSet이나 Deployment에 속한 파드들이 특정 노드에 몰려 있는 상황을 해결한다.

```
ReplicaSet "web" (파드 4개)

  Node A:  [web-1] [web-2] [web-3]   ← 3개 몰림
  Node B:  [web-4]
  Node C:  (없음)
```

kube-scheduler가 처음 배치할 당시엔 최적이었더라도, 이후 노드가 추가되거나 다른 파드가 evict되면서 불균형이 생길 수 있다. `RemoveDuplicates`는 이런 상태를 감지하고 초과분을 evict해 재스케줄링을 유도한다.

### 핵심 용어 정리

코드에 등장하는 용어를 먼저 잡고 가자.

| 용어 | 정의 |
|---|---|
| **`podOwner`** | 파드의 "소유자 + 이미지" 조합을 담는 구조체. `namespace`, `kind`, `name`, `imagesHash` 4개 필드로 구성된다. |
| **`ownerKey`** | `podOwner` 구조체 값 자체를 맵의 키로 사용할 때의 명칭. "이 파드는 어떤 owner 그룹에 속하는가"를 식별하는 단위다. |
| **`imagesHash`** | 파드의 컨테이너 이미지 목록을 정렬 후 `"#"`으로 연결한 문자열. 예: `"nginx:1.25#sidecar:v2"`. 같은 owner라도 이미지가 다르면 다른 그룹으로 취급한다. |
| **`podContainerKeys`** | 파드 하나를 `"namespace/kind/name/image"` 형태의 문자열 목록으로 표현한 것 (정렬됨). 이 목록이 같은 파드 두 개가 한 노드에 있으면 **중복**으로 판단한다. |
| **`duplicateKeysMap`** | 노드 안에서 중복을 탐지하는 임시 맵. 노드마다 초기화되므로 **같은 노드 내** 중복 탐지에만 쓰인다. |
| **`ownerKeyOccurence`** | ownerKey별 전체 파드 수. **모든 노드**를 합산한 값으로, `upperAvg` 계산에 사용된다. |
| **`duplicatePods`** | `ownerKey → 노드이름 → []*v1.Pod` 구조. 각 노드에서 중복으로 판별된 파드 목록. 각 노드의 **첫 번째 파드(기준 파드)는 포함되지 않는다.** |
| **`upperAvg`** | 균등 분배 시 노드당 허용할 최대 파드 수. `ceil(전체 파드 수 / targetNode 수)` 로 계산한다. |
| **`targetNodes`** | 파드가 실제로 스케줄 가능한 노드 목록. toleration, nodeSelector, nodeAffinity를 기준으로 필터링한다. |

### 알고리즘 개요

```
Phase 1 — 중복 탐지 (노드별 순회)
  각 노드의 파드를 순회하며:
    podContainerKeys 생성 (namespace/kind/name/image 조합, 정렬)
    같은 노드에 동일한 podContainerKeys를 가진 파드가 이미 있으면
      → duplicatePods[ownerKey][nodeName] 에 추가
    ownerKeyOccurence[ownerKey] 증가 (모든 파드 카운트)

Phase 2 — 초과분 evict (ownerKey별)
  ownerKey마다:
    targetNodes 계산 (실제 스케줄 가능한 노드들)
    upperAvg = ceil(전체 파드 수 / targetNode 수)
    각 노드에서 (중복 수 + 1) > upperAvg 이면 초과분 evict
    ※ +1 의 이유는 아래 코드 따라가기에서 설명
```

### Balance() 코드 따라가기

> `pkg/framework/plugins/removeduplicates/removeduplicates.go`

#### Phase 1: 중복 탐지

노드를 순회하면서 각 파드의 `ownerKey`와 `podContainerKeys`를 만든다. (`L110-L162`)

```go
// L132: duplicateKeysMap 은 노드마다 초기화 → 같은 노드 내 중복 탐지에만 사용
duplicateKeysMap := map[string][][]string{}

for _, pod := range pods {  // L133
    ownerRefList := podutil.OwnerRef(pod)
    if len(ownerRefList) == 0 || hasExcludedOwnerRefKind(...) {
        continue  // L136: OwnerRef 없는 bare pod, 또는 제외 대상 kind는 스킵
    }

    // L144-L145: 이미지 목록 정렬 → "#"으로 연결 → imagesHash
    sort.Strings(imageList)
    imagesHash := strings.Join(imageList, "#")

    for _, ownerRef := range ownerRefList {  // L146
        // L147-L152: ownerKey 생성 (namespace + kind + name + imagesHash)
        ownerKey := podOwner{namespace, kind: ownerRef.Kind, name: ownerRef.Name, imagesHash}
        ownerKeyOccurence[ownerKey]++  // L153: 전체 파드 수 누적 (Phase 2 upperAvg 계산에 사용)

        // L158-L159: "namespace/kind/name/image" 문자열 생성
        s := strings.Join([]string{namespace, kind, name, image}, "/")
        podContainerKeys = append(podContainerKeys, s)
    }
    sort.Strings(podContainerKeys)  // L162: 정렬해야 이후 DeepEqual 비교가 가능
    ...
}
```

`podContainerKeys`가 준비되면, 같은 노드에서 이미 같은 키를 봤는지 `duplicateKeysMap`으로 확인한다. (`L165-L193`)

```go
if existing, ok := duplicateKeysMap[podContainerKeys[0]]; ok {  // L165
    for _, keys := range existing {
        if reflect.DeepEqual(keys, podContainerKeys) {  // L168: 전체 키 목록 비교
            // 중복 발견 → duplicatePods 에 추가
            // 기준 파드(처음 본 파드)는 이미 넣지 않았으므로 여기서만 추가됨
            duplicatePods[ownerKey][node.Name] = append(..., pod)  // L181
            break
        }
    }
} else {
    // L192: 이 노드에서 처음 보는 키 → 기준 파드로 등록 (duplicatePods에는 추가하지 않음)
    duplicateKeysMap[podContainerKeys[0]] = [][]string{podContainerKeys}
}
```

`duplicateKeysMap`은 노드마다 초기화된다. 따라서 **한 노드 안에서 같은 ownerKey를 가진 파드가 2개 이상 있는 경우**만 `duplicatePods`에 기록된다. 노드 간 파드 수 불균형은 Phase 2의 `upperAvg`로 처리한다.

#### Phase 2: upperAvg 계산과 eviction

```go
for ownerKey, podNodes := range duplicatePods {  // L198
    targetNodes := getTargetNodes(ctx, podNodes, nodes)  // L200

    if len(targetNodes) < 2 {  // L203
        continue  // 갈 수 있는 노드가 1개 이하면 spread 불가 → 스킵
    }

    // L208: 균등 분배 시 노드당 최대 파드 수 (올림)
    upperAvg := int(math.Ceil(float64(ownerKeyOccurence[ownerKey]) / float64(len(targetNodes))))

    for nodeName, pods := range podNodes {  // L210
        // L212 주석: "list of duplicated pods does not contain the original referential pod"
        // pods = 중복 파드들 (기준 파드 제외)
        // 실제 이 노드의 파드 수 = len(pods) + 1  ← 기준 파드를 더해야 실제 수
        if len(pods)+1 > upperAvg {  // L213
            for _, pod := range pods[upperAvg-1:] {  // L216: upperAvg-1개는 남기고 나머지 evict
                r.handle.Evictor().Evict(ctx, pod, ...)  // L217
            }
        }
    }
}
```

예를 들어 `ownerKey`가 같은 파드가 4개이고 `targetNodes`가 3개라면:

```
ownerKeyOccurence[ownerKey] = 4
len(targetNodes) = 3
upperAvg = ceil(4 / 3) = 2

Node A: 기준 파드 1개 + 중복 파드 2개  → len(pods)+1 = 3 > 2  → pods[1:] = 1개 evict
Node B: 기준 파드 1개 + 중복 파드 0개  → duplicatePods에 없음  → 유지
Node C: (없음)                          → duplicatePods에 없음  → 유지
```

evict된 파드는 kube-scheduler에 의해 Node C에 재배치된다.

#### getTargetNodes — 실제로 갈 수 있는 노드만 계산

> `L236-L291`

`upperAvg`를 계산할 때 단순히 전체 노드 수를 쓰면 안 된다. 파드의 toleration, nodeSelector, nodeAffinity 때문에 일부 노드엔 스케줄이 불가능할 수 있기 때문이다.

```go
func getTargetNodes(...) []*v1.Node {  // L236
    // L244-L261: toleration, nodeSelector, nodeAffinity 가 동일한 파드는 중복 제거 (효율화)
    distinctPods := ...

    // L265-L283: 각 파드가 스케줄 가능한 노드만 수집
    for pod := range distinctPods {
        for _, node := range nodes {
            if !TolerationsTolerateTaintsWithFilter(...)  { continue }  // L268
            if !PodMatchNodeSelector(...)        { continue }  // L273
            targetNodesMap[node.Name] = node
        }
    }
    return targetNodes
}
```

파드가 실제로 재배치될 수 없는 노드는 분모에서 제외되어 `upperAvg`가 더 정확하게 계산된다.

## 5. [플러그인 2] RemovePodsViolatingTopologySpreadConstraint

### 문제 정의

Kubernetes의 `TopologySpreadConstraint(TSC)`는 파드를 토폴로지 도메인(예: 가용 영역, 노드)에 고르게 분산시키는 규칙이다. 이 규칙이 처음엔 만족되었더라도, 노드 추가/제거나 다른 eviction 이후 불균형이 생길 수 있다.

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

`RemovePodsViolatingTopologySpreadConstraint`는 이런 위반 상태를 감지하고 초과 도메인에서 파드를 evict해 재분배를 유도한다.

### 핵심 용어 정리

| 용어 | 정의 |
|---|---|
| **`TopologySpreadConstraint (TSC)`** | 파드 분산 규칙. `topologyKey`로 도메인을 나누고 `maxSkew`로 허용 불균형을 지정한다. |
| **`topologyKey`** | 도메인 분류 기준이 되는 노드 레이블 키. 예: `"topology.kubernetes.io/zone"`. |
| **`maxSkew`** | 각 도메인의 파드 수와 전역 최솟값 사이의 허용 최대 차이. 예: maxSkew=1이면 어떤 도메인도 가장 적은 도메인보다 2개 이상 많을 수 없다. |
| **`topologyPair`** | `{key, value}` 쌍으로 하나의 토폴로지 도메인을 식별한다. 예: `{key: "zone", value: "us-east-1a"}`. |
| **`topology`** | `topologyPair` + 해당 도메인에 속한 파드 목록. `balanceDomains()`에서 도메인 단위로 다룬다. |
| **`constraintTopologies`** | 하나의 TSC에 대해 도메인별 파드 목록을 담는 맵. `topologyPair → []*v1.Pod`. |
| **`sortedDomains`** | `constraintTopologies`를 파드 수 기준 오름차순으로 정렬한 `[]topology`. two-pointer 알고리즘의 입력이다. |
| **`idealAvg`** | 이상적인 도메인당 평균 파드 수. `sumPods / len(constraintTopologies)`로 계산한다. |
| **`podsForEviction`** | 모든 TSC 처리 결과를 모은 최종 evict 대상 파드 집합. 중복 eviction 방지를 위해 맵으로 관리한다. |

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

먼저 클러스터 전체 파드를 네임스페이스별로 묶는다. (`L138-L147`)

```go
pods, err := podutil.ListPodsOnNodes(nodes, ...)  // L138: 전체 노드의 파드 수집
namespacedPods := podutil.GroupByNamespace(pods)   // L147: 네임스페이스별 그룹화
```

각 네임스페이스에서 고유한 TSC를 수집한다. (`L154-L176`)

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

TSC마다 `constraintTopologies`를 초기화할 때, **파드가 0개인 도메인도 미리 등록**한다. (`L183-L192`)

```go
// L186-L192: 해당 topologyKey 레이블을 가진 모든 노드를 빈 슬라이스로 초기화
for _, node := range nodeMap {
    if val, ok := node.Labels[tsc.TopologyKey]; ok {
        constraintTopologies[topologyPair{key: tsc.TopologyKey, value: val}] = make([]*v1.Pod, 0)
    }
}
```

빈 도메인을 미리 넣는 이유: `idealAvg` 계산 시 파드가 없는 도메인도 분모에 포함해야 올바른 평균이 나오기 때문이다.

이후 각 파드를 해당 topologyPair에 할당한다. (`L197-L222`)

```go
for _, pod := range namespacedPods[namespace] {  // L197
    if !tsc.Selector.Matches(labels.Set(pod.Labels)) { continue }  // L203: 레이블 셀렉터 불일치 skip
    nodeValue := node.Labels[tsc.TopologyKey]
    topoPair := topologyPair{key: tsc.TopologyKey, value: nodeValue}  // L218
    constraintTopologies[topoPair] = append(..., pod)  // L220
    sumPods++
}
```

이미 균형이면 `balanceDomains()` 호출을 건너뛴다. (`L223`)

```go
if topologyIsBalanced(constraintTopologies, tsc) { continue }  // L223
d.balanceDomains(ctx, podsForEviction, tsc, constraintTopologies, sumPods, nodes)  // L227
```

#### Phase 2: balanceDomains() — two-pointer

모든 TSC 처리가 끝나면 `podsForEviction`을 실제로 evict한다. 이 집합을 채우는 핵심 로직이 `balanceDomains()`다.

```go
func (d *...) balanceDomains(...) {  // L308
    idealAvg := sumPods / float64(len(constraintTopologies))  // L309
    sortedDomains := sortDomains(constraintTopologies, ...)   // L311: 파드 수 오름차순 정렬
    ...
    i := 0                        // L320: 가장 작은 도메인 포인터
    j := len(sortedDomains) - 1   // L321: 가장 큰 도메인 포인터
```

`sortDomains()`는 도메인을 파드 수 오름차순으로 정렬하고, 각 도메인 안의 파드도 eviction 우선순위 순으로 정렬한다. 슬라이스 **뒤쪽**이 우선 evict 대상이며, 뒤쪽부터 순서는 다음과 같다.

```
← 보호 (앞)                                         evict 우선 (뒤) →
  non-evictable | 셀렉터/affinity 있음 낮은→높은 우선순위 | 셀렉터/affinity 없음 낮은→높은 우선순위
```

즉, **셀렉터/affinity가 없고 우선순위가 높은 파드**가 가장 먼저 evict 대상이 된다.

two-pointer 루프: (`L322-L386`)

```go
for i < j {  // L322
    // j 도메인이 idealAvg 이하면 더 줄 수 없음 → j를 당김
    if float64(len(sortedDomains[j].pods)) <= idealAvg {  // L324
        j--; continue
    }

    skew := float64(len(sortedDomains[j].pods) - len(sortedDomains[i].pods))  // L329

    // skew가 maxSkew 이내면 이 쌍은 허용 범위 → i를 올림
    if int32(skew) <= tsc.MaxSkew {  // L332
        i++; continue
    }

    // 이동할 파드 수 계산 (세 가지 제약 중 최솟값)
    aboveAvg  := math.Ceil(float64(len(sortedDomains[j].pods)) - idealAvg)  // L343: j가 평균 초과분
    belowAvg  := math.Ceil(idealAvg - float64(len(sortedDomains[i].pods)))  // L344: i가 평균 미달분
    halfSkew  := math.Ceil((skew - float64(tsc.MaxSkew)) / 2)               // L346: skew 절반
    movePods  := int(math.Min(math.Min(aboveAvg, belowAvg), halfSkew))      // L347

    if movePods <= 0 { i++; continue }  // L348

    // j의 뒤쪽 movePods개를 podsForEviction에 추가  (L362-L383)
    aboveToEvict := sortedDomains[j].pods[len(sortedDomains[j].pods)-movePods:]
    for k := range aboveToEvict {
        podsForEviction[aboveToEvict[k]] = struct{}{}  // L382
    }
    // 도메인 파드 수 갱신 (알고리즘 진행용 추적)
    sortedDomains[j].pods = sortedDomains[j].pods[:len(sortedDomains[j].pods)-movePods]  // L384
    sortedDomains[i].pods = append(sortedDomains[i].pods, aboveToEvict...)               // L385
}
```

예시 (`maxSkew=1`, 도메인 6개):

```
초기 정렬: [2, 3, 5, 5, 7, 8]  idealAvg = 30/6 = 5

i=0(2), j=5(8): skew=6 > 1, aboveAvg=3, belowAvg=3, halfSkew=3 → movePods=3
  [5, 3, 5, 5, 7, 5]

i=0(5), j=5(5): j ≤ idealAvg → j--
i=0(5), j=4(7): skew=2 > 1, aboveAvg=2, belowAvg=0 → movePods=0 → i++

i=1(3), j=4(7): skew=4 > 1, aboveAvg=2, belowAvg=2, halfSkew=2 → movePods=2
  [5, 5, 5, 5, 5, 5]

i=1(5), j=4(5): j ≤ idealAvg → j-- ... i >= j → 종료
```

#### Phase 3: 실제 eviction

`podsForEviction`에 모인 파드를 순회하며 evict한다. (`L231-L254`)

```go
for pod := range podsForEviction {  // L232
    if nodeLimitExceeded[pod.Spec.NodeName] { continue }  // L233: 노드 한도 초과 시 skip
    if !d.podFilter(pod) { continue }                     // L236: Filter 재확인

    if d.handle.Evictor().PreEvictionFilter(pod) {        // L240: 최종 확인 (NodeFit 등)
        err := d.handle.Evictor().Evict(ctx, pod, ...)    // L241
        ...
    }
}
```

`RemoveDuplicates`와 달리, TSC 플러그인은 eviction을 두 단계로 분리한다. `balanceDomains()`에서는 **후보만 선정**하고, 실제 eviction은 모든 TSC 처리가 끝난 뒤 한꺼번에 수행한다. 이렇게 하면 여러 TSC가 동일한 파드를 두 번 evict하는 것을 방지할 수 있다.

## 6. 정리 및 비교

### 두 플러그인 비교

두 플러그인 모두 `BalancePlugin`을 구현하지만, 감지하는 문제와 해결 방식이 다르다.

#### 문제 정의

| | RemoveDuplicates | RemovePodsViolatingTopologySpreadConstraint |
|---|---|---|
| **감지 대상** | 같은 owner의 파드가 한 노드에 2개 이상 몰린 경우 | TSC의 maxSkew를 초과한 도메인 간 불균형 |
| **범위** | 노드 단위 | 토폴로지 도메인 단위 (zone, region 등) |
| **트리거 조건** | 명시적 규칙 없이 소유자 기준 자동 감지 | 파드에 TSC가 명시되어 있어야 동작 |

#### 감지 방식

| | RemoveDuplicates | RemovePodsViolatingTopologySpreadConstraint |
|---|---|---|
| **집계 단위** | 노드별 `podContainerKeys` 비교 | 네임스페이스별 TSC 수집 → 도메인별 파드 수 집계 |
| **빈 버킷 처리** | `duplicatePods`에 기록 없음 (Phase 2에서 처리) | `constraintTopologies` 초기화 시 빈 도메인도 미리 등록 |
| **기준 계산** | `upperAvg = ceil(전체 파드 수 / targetNode 수)` | `idealAvg = 전체 파드 수 / 도메인 수` |

#### 알고리즘

| | RemoveDuplicates | RemovePodsViolatingTopologySpreadConstraint |
|---|---|---|
| **핵심 알고리즘** | 단순 임계값 비교 (`len(pods)+1 > upperAvg`) | two-pointer on sorted domains |
| **eviction 시점** | 초과분 발견 즉시 evict | 모든 TSC 처리 후 `podsForEviction` 한꺼번에 evict |
| **즉시 evict 이유** | ownerKey별 독립 처리, 중복 eviction 위험 없음 | 여러 TSC가 동일 파드를 중복 선정할 수 있어 일괄 처리 필요 |

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

#### eviction은 재스케줄링을 보장하지 않는다

Descheduler는 파드를 evict할 뿐, 어디로 재배치될지는 kube-scheduler가 결정한다. evict 후 동일한 노드로 돌아올 수도 있다. 특히 `NodeFit` 옵션이 꺼져 있으면 갈 곳 없는 파드도 evict된다.

```
권장: DefaultEvictor의 NodeFit 옵션 활성화
→ PreEvictionFilter 단계에서 다른 노드에 스케줄 가능한지 확인 후 evict
```

#### PDB 설정 필수

PodDisruptionBudget(PDB)이 없으면 Descheduler가 동시에 너무 많은 파드를 evict해 서비스 중단이 발생할 수 있다. `PodEvictor`는 eviction 전 PDB를 확인하므로, 중요한 워크로드에는 반드시 PDB를 설정해야 한다.

#### eviction 한도 설정

Policy에서 3가지 한도를 조합해 과도한 eviction을 방지한다.

```yaml
maxNoOfPodsToEvictPerNode: 5        # 노드당 한 루프에 최대 eviction 수
maxNoOfPodsToEvictPerNamespace: 10  # 네임스페이스당 한 루프에 최대 eviction 수
maxNoOfPodsToEvictTotal: 50         # 전체 한 루프에 최대 eviction 수
```

#### 실행 전 dry-run 확인

Descheduler는 `--dry-run` 플래그를 지원한다. 실제 eviction 없이 어떤 파드가 evict 대상이 되는지 로그로 확인할 수 있다. 운영 환경 적용 전에 반드시 dry-run을 먼저 실행하자.

#### Deployment 모드의 interval 설정

interval이 너무 짧으면 이전 eviction으로 인한 재스케줄링이 완료되기 전에 다음 루프가 돌아 불필요한 eviction이 반복될 수 있다. 클러스터 규모와 kube-scheduler의 처리 속도를 고려해 충분한 interval을 설정해야 한다.

---

## 7. 마무리

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

| 구성 요소 | 역할 | 핵심 메커니즘 |
|---|---|---|
| `RemoveDuplicates` | 같은 owner의 파드가 특정 노드에 몰린 경우 해소 | `upperAvg` 기준 초과분 즉시 evict |
| `RemovePodsViolatingTopologySpreadConstraint` | TSC `maxSkew` 위반 상태 해소 | two-pointer로 최소 이동 evict 후보 선정 |
| `DefaultEvictor` | evict 부작용 방지 | NodeFit · PDB 확인 후 최종 승인 |
| **Descheduler 전체** | kube-scheduler의 일회성 배치 결정을 주기적으로 재검토 | 매 루프마다 플러그인을 실행해 최적 상태 유지 |

> **kube-scheduler는 "지금 최선"을 고른다. Descheduler는 "지금도 최선인지"를 주기적으로 되묻는다.**

---

## 부록: 주요 코드 경로

| 항목 | 경로 |
|---|---|
| 진입점 | `cmd/descheduler/descheduler.go` |
| 서버 초기화 | `cmd/descheduler/app/server.go` |
| 핵심 루프 | `pkg/descheduler/descheduler.go` |
| 플러그인 인터페이스 | `pkg/framework/types/types.go` |
| Profile 초기화 | `pkg/framework/profile/profile.go` |
| DefaultEvictor | `pkg/framework/plugins/defaultevictor/defaultevictor.go` |
| Eviction 처리 | `pkg/descheduler/evictions/evictions.go` |
| RemoveDuplicates | `pkg/framework/plugins/removeduplicates/removeduplicates.go` |
| TSC 플러그인 | `pkg/framework/plugins/removepodsviolatingtopologyspreadconstraint/topologyspreadconstraint.go` |
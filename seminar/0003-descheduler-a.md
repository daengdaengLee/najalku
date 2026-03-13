# Descheduler: 플러그인 아키텍처와 RemoveDuplicates
## kube-scheduler가 놓친 자리를 채우는 법 (Part A)

> **대주제:** k8s 클러스터 운영
> **중주제:** Descheduler
> **발표 시간:** 약 15~20분

---

## 목차

1. 왜 Descheduler가 필요한가?
2. 전체 동작 흐름
3. Plugin 시스템
4. RemoveDuplicates
5. 다음 시간 예고
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

## 4. RemoveDuplicates

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

## 5. 다음 시간 예고

다음 세미나(Part B)에서는 두 번째 플러그인인 `RemovePodsViolatingTopologySpreadConstraint`를 다룬다. TSC의 `maxSkew` 위반을 two-pointer 알고리즘으로 해소하는 방식을 코드 레벨로 살펴보고, `RemoveDuplicates`와의 설계 차이를 비교한다.

- **RemovePodsViolatingTopologySpreadConstraint**: 가용 영역·노드 등 토폴로지 도메인 간 파드 수 불균형을 감지하고, 최소 이동 수로 위반을 해소하는 two-pointer 알고리즘
- **두 플러그인 비교**: 감지 단위(노드 vs 도메인), 기준값 계산(`upperAvg` vs `idealAvg`), eviction 시점(즉시 vs 일괄)의 차이

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

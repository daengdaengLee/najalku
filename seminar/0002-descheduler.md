# Descheduler: Kubernetes 파드 재배치 전략

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

#### 호출 스택

```
main()                                            # cmd/descheduler/descheduler.go
└─ app.NewDeschedulerCommand()                    # cmd/descheduler/app/server.go
   ├─ PreRunE: SetupPlugins()                     # 플러그인 레지스트리 등록
   └─ RunE: app.Run()
      └─ descheduler.Run()                        # pkg/descheduler/descheduler.go
         ├─ LoadPolicyConfig()                    # 설정 파일 파싱
         └─ RunDeschedulerStrategies()
            ├─ SharedInformerFactory 생성
            ├─ bootstrapDescheduler()
            │  ├─ NewPodEvictor()                 # eviction 실행기 생성
            │  ├─ Profile별 Plugin 인스턴스화
            │  └─ sharedInformerFactory.Start()
            │     WaitForCacheSync()              # 캐시 동기화 완료까지 대기
            └─ wait.NonSlidingUntil(runLoop, interval, stopCh)
               └─ runDeschedulerLoop()
                  └─ runProfiles()
                     ├─ 모든 Profile의 Deschedule 플러그인 실행
                     └─ 모든 Profile의 Balance 플러그인 실행
```

#### 실행 모드

Descheduler는 세 가지 방식으로 배포할 수 있다.

| 배포 방식 | `DeschedulingInterval` | 동작 |
|---|---|---|
| **Job** | `0` | 한 번 실행 후 종료. 외부에서 직접 실행 |
| **CronJob** | `0` | 한 번 실행 후 종료. Kubernetes CronJob이 주기적으로 다시 실행 |
| **Deployment** | `> 0` | 프로세스가 계속 살아있으며 interval마다 반복 실행 |

Job과 CronJob의 차이는 Descheduler 코드 내부에는 없고, **어떻게 배포하느냐**의 차이다. CronJob은 Kubernetes가 스케줄에 맞춰 새 Job을 띄워주는 방식이고, Deployment는 프로세스 자체가 루프를 돌며 interval을 관리한다.

`wait.NonSlidingUntil`은 이전 실행 **시작** 시점부터 interval을 센다. 실행 시간이 interval을 초과하면 종료 직후 바로 다음 실행을 시작한다.

interval이 `0`이면 한 번 실행 후 context를 cancel해서 루프를 종료한다 (`pkg/descheduler/descheduler.go:546-548`).

#### SharedInformerFactory — 왜 캐시를 쓰는가?

플러그인들은 매 루프마다 클러스터의 노드/파드 목록을 조회한다. 매번 API 서버에 직접 요청하면 부하가 크기 때문에, client-go의 **Informer**를 사용해 로컬 캐시에서 읽는다.

```
API 서버 --Watch--> Informer (로컬 캐시) --List--> 플러그인
```

`WaitForCacheSync()`는 첫 루프가 시작되기 전, 캐시가 완전히 채워질 때까지 블로킹한다.

#### runProfiles: Deschedule → Balance 순서

```go
// 먼저 모든 Profile의 Deschedule 플러그인 실행
for _, profileR := range d.profileRunners {
    profileR.descheduleEPs(ctx, nodes)
}

// 그 다음 모든 Profile의 Balance 플러그인 실행
for _, profileR := range d.profileRunners {
    profileR.balanceEPs(ctx, nodes)
}
```

Deschedule이 먼저 실행되어 "명백한 문제(규칙 위반)" 파드를 먼저 제거하고, Balance가 나중에 실행되어 남은 파드들을 재분배한다. 이번 세미나에서 다루는 `RemoveDuplicates`와 `RemovePodsViolatingTopologySpreadConstraint`는 모두 **Balance 플러그인**이다.
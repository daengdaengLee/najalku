# Cilium 아키텍처 — Agent · Operator · Hubble · CNI 의 자리와 책임

이 문서는 Cilium 시리즈의 세 번째 자료입니다. 이전 세션(0008 eBPF 훅)은 XDP, tc, socket 세 훅이 커널 네트워크 스택의 어디에 부착되고 각각 어떤 컨텍스트와 결정 권한을 가지는지를 정리했습니다. 이 문서는 그다음 질문, 즉 "누가 그 훅에 BPF 프로그램을 부착하고 누가 BPF map 을 갱신하는가" 에 답하기 위해 Cilium 의 컴포넌트들을 클러스터 다이어그램 위에 올려놓고 각 컴포넌트가 어느 자리에서 무엇을 책임지는지를 정리합니다.

## 문제 — 0008 이 남긴 질문

0008 은 훅이 어디에 끼어드는지를 답했지만, 그 훅에 BPF 프로그램을 부착하고 BPF map 을 갱신하는 주체가 누구인지는 다루지 않았습니다. 그러나 그 주체가 노드 로컬인지 클러스터 범위인지에 따라 책임의 자리가 달라지고, 클러스터 전체의 동작을 그릴 수 있는 좌표계도 달라집니다. 따라서 Cilium 을 컴포넌트 단위로 이해하려면 노드 위에서 datapath 를 다루는 주체와 노드를 넘는 상태를 다루는 주체를 먼저 분리하고, 그 위에 관찰성과 CNI 결합점을 얹어야 합니다.

## 핵심 한 줄

Cilium 은 노드당 하나의 `cilium-agent` DaemonSet, 클러스터당 하나의 `cilium-operator` Deployment, 그리고 그 위에 노드 로컬 observer 와 클러스터 단위 Relay 의 쌍으로 동작하는 Hubble 로 구성됩니다. 0008 의 세 훅에 BPF 프로그램을 부착하고 BPF map 을 갱신하는 주체는 모두 노드 위의 Agent 이며, Operator 는 그 datapath 에 직접 손대지 않고 클러스터 범위 상태만 다룹니다.

## 한 그림으로 보는 컴포넌트 토폴로지

컴포넌트별 책임을 살펴보기 전에 클러스터 위의 자리를 먼저 정리합니다.

```
   ─── 클러스터 단위 (노드 무관) ───            ─── 노드 로컬 (워커 노드 N 개) ───

   ┌──────────────────────┐                   ┌────────────────────────────────┐
   │   kube-apiserver     │ ◀── watch ────────│  Node 1                        │
   └──────────┬───────────┘                   │   cilium-agent (DaemonSet)     │
              ▲                               │     + Hubble observer          │
              │ CR R/W                        │   cilium-cni 바이너리          │
              │                               │   호스트 커널 BPF · BPF map     │
   ┌──────────┴───────────┐                   │   Pods (veth 부착)             │
   │   cilium-operator    │                   └────────────────────────────────┘
   │   Deployment         │
   │   (leader election)  │                   ┌────────────────────────────────┐
   └──────────────────────┘                   │  Node 2                        │
                                              │   (Node 1 과 동일 구성)        │
   ┌──────────────────────┐ ◀── gRPC ─────────└────────────────────────────────┘
   │   hubble-relay       │   Relay → observer 연결
   │   Deployment         │   플로우 흐름은 observer → Relay
   │   (gRPC client)      │
   ├──────────────────────┤                   ┌────────────────────────────────┐
   │   hubble-ui          │                   │  Node N                        │
   │   Deployment         │                   │   ...                          │
   └──────────────────────┘                   └────────────────────────────────┘
```

분할 기준은 "컨트롤플레인 vs 워커" 가 아니라 "클러스터 단위 vs 노드 로컬" 입니다. `cilium-operator` 와 `hubble-relay` Deployment 는 워커 노드에 스케줄링될 수도 있고, 사용자가 별도로 지정하지 않는 한 자리 자체는 노드 스케줄러가 결정합니다. 노드 로컬 면의 모든 화살표가 결국 `kube-apiserver` 의 watch 스트림으로 모인다는 점은 다음 절에서 단일 이벤트의 흐름으로 다시 확인합니다.

## 메커니즘 — 컴포넌트별 자리와 책임

위 좌표계 안에서 컴포넌트 하나하나가 무엇을 책임지는지를 살펴봅니다. 0008 의 세 훅에 프로그램을 부착하고 BPF map 을 갱신하는 일이 어디에서 일어나는지부터 시작합니다.

### Agent — 노드당 DaemonSet, datapath 의 주인

`cilium-agent` 는 노드당 한 개의 Pod 가 보장되도록 `DaemonSet` 으로 배포됩니다. 호스트 커널의 BPF 서브시스템에 직접 접근해야 하므로 `hostNetwork` 와 `privileged` 권한을 가지고 실행되며, 그 결과 BPF 프로그램의 컴파일·로드·부착과 BPF map 의 소유·갱신이 모두 이 Agent 안에서 일어납니다. 한 노드의 datapath 에서 일어나는 거의 모든 결정의 주체가 이 자리입니다.

Agent 는 `kube-apiserver` 를 informer 로 watch 합니다. `Service`, `EndpointSlice`, `Pod`, `NetworkPolicy`, `CiliumIdentity` 같은 리소스의 변화가 들어오면 내부의 datapath manager 가 그 변화를 BPF map 에 반영합니다. 사용자공간에서 BPF map 항목을 갱신하는 호출(`bpf_map_update_elem` 계열)이 이 자리에서 일어나고, 그 결과가 곧 노드 위에서 패킷이 실제로 어떻게 분류·전달·차단될지를 결정합니다. 같은 watch 이벤트는 노드별 Agent 에 독립적으로 도달하고 각자 자기 노드의 BPF map 을 갱신하므로, 노드 간 정합성은 `kube-apiserver` 가 단일 진실을 들고 있다는 점에서만 보장됩니다.

BPF 프로그램의 빌드와 부착도 Agent 의 일입니다. Agent 는 BPF C 소스를 컴파일해 ELF 바이트코드를 만들고 `bpf()` 시스템 콜로 커널에 로드합니다. 0007 에서 정리한 대로 이 로드 시점에 커널 verifier 가 안전성을 정적으로 검증하고, 그 검증을 통과한 프로그램만 `XDP`, `tc`, `cgroup` 의 hook 자리에 부착됩니다. 즉 0008 의 세 훅에 들어가는 프로그램은 모두 같은 Agent 가 같은 노드 위에서 직접 부착한 결과입니다. Agent 는 또한 노드 위의 각 Pod 를 Cilium 의 Endpoint 추상으로 들고 identity 와 묶어 관리하며, 정책 컴파일과 BPF policy map 주입도 모두 이 자리의 책임입니다. 0006 에서 kube-proxy 가 노드당 하나의 DaemonSet 으로 `KUBE-*` 체인을 관리하던 자리에 그대로 Agent 가 들어와 있고, 다만 그 보관소가 iptables 체인이 아니라 BPF map 으로 바뀌어 있다고 보면 위치가 명확합니다.

CNI 플러그인은 별도의 컴포넌트라기보다 이 Agent 의 위성으로 보아야 자리가 잘 잡힙니다. kubelet 이 새 Pod 를 만들 때 호출하는 것은 노드에 설치된 `cilium-cni` 바이너리이지만, 이 바이너리는 IPAM 이나 Endpoint 객체 생성 같은 결정을 스스로 하지 않고 Unix socket 으로 Agent 의 로컬 API 에 그대로 위임합니다. 그 결과 IP 할당과 Endpoint 등록은 Agent 가 수행하고, `cilium-cni` 는 veth 페어 생성과 tc hook 부착 같은 노드 측 네트워크 셋업의 손발 노릇만 합니다. 따라서 CNI 는 Agent 의 입구일 뿐이고, 한 노드에서 일어나는 모든 datapath 변경은 결국 Agent 하나로 수렴합니다.

다음 그림은 이 노드 하나를 확대해서 본 모습입니다.

```
                  ─── Worker Node ───

   사용자공간                                          커널공간
   ─────────────────────────                          ──────────────────────────

    [kubelet]
       │ CNI ADD/DEL
       ▼
    [cilium-cni 바이너리]
       │ 위임 (Unix socket)
       ▼
   ┌─ cilium-agent Pod (hostNetwork, privileged) ─┐
   │                                              │
   │  k8s informer ── watch ─────▶ kube-apiserver │
   │                                              │
   │  datapath manager  ┐                         │
   │  Endpoint 관리자   ├─ 키-값 write ───────────┼─▶  [BPF map 집합]
   │  정책 컴파일러     ┘                         │         ▲      │
   │                                              │         │ lookup
   │  BPF loader ───── bpf() load + attach ───────┼─▶  [hook 자리]
   │                                              │      XDP
   │  Hubble observer ◀── perf / ringbuffer ──────┼─    tc ingress/egress
   │                                              │      socket (cgroup v2)
   │  local API                                   │
   └──────────────────────────────────────────────┘

                                       [Pod] ── veth ── [lxc<hex>]
                                                         (호스트 측, tc 부착)
```

한 노드 위의 모든 화살표가 결국 Agent Pod 한 곳에서 출발하거나 그곳으로 모이고, BPF 프로그램과 BPF map 은 모두 호스트 커널에 있지만 그 주인은 변함없이 Agent 입니다.

### Operator — 클러스터당 하나, 노드를 넘는 상태의 관리자

`cilium-operator` 는 한 노드의 datapath 가 결정할 수 없는 클러스터 범위 상태를 다루기 위해 자리가 분리된 컴포넌트입니다. Helm 기본값 기준 보통 1~2 replica 의 `Deployment` 로 떠 있고, replica 가 둘 이상이면 leader election 으로 active 인스턴스 하나만 일을 합니다. 노드 스케줄링은 컨트롤플레인 노드에 묶여 있지 않으며, 클러스터 어디에 떠 있든 책임이 클러스터 단위라는 점이 자리의 본질입니다.

Operator 는 호스트 커널의 BPF 에 손대지 않습니다. BPF 프로그램을 부착하거나 BPF map 을 갱신하는 일은 Agent 의 몫이고, Operator 는 그 일을 가능하게 하는 클러스터 단위 상태만 정돈합니다. 대표적으로 `CiliumNode` 재고를 클러스터 전역으로 조정해 각 노드에 PodCIDR 이나 IPAM 풀을 분배하고, IPAM 모드(cluster-pool, ENI, Azure 등)별로 다른 외부 자원 할당 작업을 수행합니다. 또한 더 이상 사용되지 않는 `CiliumIdentity` 의 garbage collection 과 CRD 등록·마이그레이션도 이 자리에서 진행합니다. 즉 Operator 가 클러스터 단위 reconcile 루프를 돌리는 동안, 노드별 Agent 는 그 결과를 자기 노드의 datapath 에 반영하는 별도의 루프를 돕니다. 자리상으로는 Kubernetes 의 `kube-controller-manager` 가 컨트롤플레인에서 클러스터 범위 reconcile 을 담당하던 패턴과 그대로 평행합니다.

"Agent 는 노드 로컬, Operator 는 클러스터 범위" 라는 분담선은 Cilium 의 컴포넌트 토폴로지에서 가장 큰 분할선이고, 이후 NetworkPolicy 와 identity 를 다루는 세션들도 같은 선 위에서 책임이 나뉩니다.

### Hubble — Agent 위의 관찰성 면

Hubble 은 Cilium 의 별도 컴포넌트라기보다 노드 로컬 observer 와 클러스터 단위 Relay 가 한 쌍을 이루는 관찰성 면입니다. observer 는 `cilium-agent` Pod 안에 함께 들어 있어 BPF 프로그램이 perf event 와 ringbuffer 로 내보내는 플로우를 같은 노드의 사용자공간에서 1차 수신합니다. Relay 는 `hubble-relay` Deployment 로 클러스터 한 곳에 떠 있으면서 각 노드의 observer 에 gRPC 클라이언트로 접속해 플로우를 한곳으로 모으고, `hubble-ui` Deployment 와 CLI(`hubble observe`), Prometheus exporter 가 그 결과를 소비합니다. 0007 마무리에서 미뤘던 "디버깅 난이도" 의 응답이 바로 이 자리에 해당하고, 플로우 이벤트가 BPF 에서 사용자에게 도달하기까지의 내부 파이프라인은 별도의 Hubble 세션에서 다룹니다.

### 한 이벤트의 흐름 — kube-apiserver → Agent → BPF map

지금까지 정의한 자리들이 실제로 어떻게 협력하는지는 단일 이벤트 하나로 압축할 때 가장 분명합니다. 예를 들어 `Service` 한 건이 새로 생성되면 그 변화는 다음과 같은 직선 경로로 노드의 BPF map 까지 도달합니다.

```
   [kubectl apply -f svc.yaml]
              │
              ▼
       kube-apiserver         (etcd 단일 진실)
              │
              │ watch 이벤트 (모든 노드 Agent 가 각자 수신)
              ▼
   ┌─ 노드 K 의 cilium-agent ─────────────────┐
   │                                          │
   │   k8s informer                           │
   │        │                                 │
   │        ▼                                 │
   │   Service handler                        │
   │        │                                 │
   │        ▼                                 │
   │   datapath manager                       │
   │        │  bpf_map_update_elem            │
   │        ▼                                 │
   │   Service BPF map                        │
   └──────────────────────────────────────────┘
```

여기서 두 가지를 짚어둘 만합니다. 첫째, 같은 watch 이벤트는 노드별 Agent 에 독립적으로 도달해서 각자 자기 노드의 BPF map 을 갱신합니다. 노드 간 정합성을 별도의 동기화 채널이 보장하는 것이 아니라, `kube-apiserver` 가 들고 있는 단일 상태가 모든 Agent 에 같은 모양으로 흘러들어가는 것입니다. 둘째, Operator 는 이 데이터 경로 위에 없습니다. Operator 는 같은 `kube-apiserver` 를 보고 별도의 클러스터 범위 reconcile 루프를 돌릴 뿐, Agent 와 BPF map 사이의 직선 경로에는 끼지 않습니다. 이 직선의 코드 수준 추적은 이후 코드 분석 세션에서 다시 다룹니다.

## 마무리

### 자리와 책임의 분배

컴포넌트별 책임을 한 줄씩으로 압축하면 본 유닛의 좌표계가 그대로 드러납니다.

- Agent: 노드 로컬, BPF 프로그램 부착과 BPF map 소유·갱신, CNI 위성을 통한 Pod 네트워킹 셋업.
- Operator: 클러스터 범위, 노드 재고·IPAM 풀·identity GC·CRD 등 노드를 넘는 상태의 reconcile.
- Hubble: Agent 위의 관찰성 면, 노드 observer 와 클러스터 Relay 의 쌍.

0008 의 세 훅이 같은 패킷에 대한 책임을 자리별로 나눠 가졌듯이, Cilium 의 컴포넌트도 같은 클러스터 상태에 대한 책임을 자리별로 나눠 가집니다.

### 본 세션이 답하지 않은 것

본 문서는 컴포넌트들의 자리와 책임을 좌표계로 그렸을 뿐, 그 자리 안에서 일어나는 일을 코드 수준으로 추적하지는 않았습니다. Agent 의 informer 가 BPF map 까지 도달하는 종단간 코드 경로는 이후 Service LB 세션과 Agent 코드 분석 세션에서, BPF C 프로그램이 패킷을 어떻게 처리하는지는 BPF 코드 분석 세션에서, identity 의 의미와 정책 결정의 흐름은 NetworkPolicy 세션에서, Hubble 의 내부 파이프라인은 별도의 Hubble 세션에서 다룹니다. Operator 의 leader election 과 IPAM 모드별 상세 동작은 본 시리즈의 선택적 확장으로 남겨둡니다.

### 다음 다리 — Cilium Service LB (ClusterIP)

본 문서는 Agent 에 "BPF map 갱신" 이라는 굵은 책임을 부여하고 거기서 멈췄습니다. 다음 세션은 그 갱신을 ClusterIP 단일 시나리오로 확대해, 0006 에서 본 `KUBE-SERVICES → KUBE-SVC-* → KUBE-SEP-*` 의 점프 경로가 `cilium_lb4_services` 와 `cilium_lb4_backends` 라는 BPF map 의 키-값 구조로 어떻게 1:1 대응되는지를 살펴봅니다.

## 부록 — 컴포넌트별 코드 진입점

본 시리즈의 이후 코드 분석 세션에서 따라갈 진입점을 컴포넌트별로 정리해 둡니다. 경로는 Cilium 저장소 루트를 기준으로 합니다.

| 컴포넌트 | 1차 진입점 | 보조 경로 | 본 유닛에서의 역할 |
| --- | --- | --- | --- |
| Agent | `daemon/main.go` | `daemon/cmd/`, `pkg/datapath/`, `pkg/endpoint/`, `pkg/k8s/watchers/` | 노드 로컬 datapath 의 주인 |
| Operator | `operator/cmd/` | `operator/pkg/`, `pkg/ipam/`, `pkg/identity/` | 클러스터 범위 reconcile |
| CNI (Agent 위성) | `plugins/cilium-cni/` | Agent 로컬 API 핸들러 | kubelet 의 위임을 Agent 로 전달 |
| Hubble | `pkg/hubble/` | `hubble-relay/`, `api/v1/observer/` | observer 와 Relay 의 쌍 |

각 경로의 종단간 추적은 이후 코드 분석 세션에서 다룹니다.

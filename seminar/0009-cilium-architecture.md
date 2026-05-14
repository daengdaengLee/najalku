# Cilium 아키텍처 — Agent / Operator / CNI / Hubble 의 자리와 책임

이 문서는 Cilium 시리즈의 세 번째 자료입니다. 이전 세션(0008 eBPF 훅)은 Cilium 이 사용하는 세 훅(XDP / tc / socket)이 커널 네트워크 스택의 어느 위치에 부착되고 어떤 권한을 갖는지를 다루면서, 다음 질문을 남겼습니다. 누가 그 훅에 BPF 프로그램을 부착하고, 누가 k8s 이벤트에 반응해 BPF map 을 갱신하는가. 이 문서는 그 주체에 해당하는 Cilium 의 컴포넌트 토폴로지(Agent / Operator / CNI / Hubble)를 클러스터 다이어그램 위에 배치하고 각 컴포넌트의 책임을 정리합니다.

## 문제 — 0008 이 남긴 질문

0007 은 BPF map 과 hook 의 메커니즘을, 0008 은 세 훅의 부착 위치와 결정 권한을 다뤘습니다. 그러나 BPF 프로그램이 hook 에 저절로 부착되지는 않습니다. 사용자공간의 누군가가 `bpf()` 시스템 콜로 BPF 프로그램을 커널에 로드하고, attach 호출로 hook 에 묶어야 합니다. k8s 에서 Service 나 NetworkPolicy 가 바뀔 때마다 BPF map 도 누군가가 갱신해야 클러스터 상태와 일치합니다.

동시에 Cilium 은 데이터플레인 제어만 담당하지 않습니다. 새 Pod 가 생성될 때 veth 페어를 만들고 IP 를 할당하는 역할(CNI), 클러스터 전체를 대상으로 identity 를 정리하고 IPAM 을 코디네이션하는 역할(Operator), 패킷 흐름을 관찰하고 외부에 노출하는 역할(Hubble)이 모두 같은 인프라 위에서 함께 떠 있어야 합니다. 이들이 어떤 형태로 배포되고 서로 어떻게 협력하는지를 한 그림으로 잡는 것이 이 문서의 목표입니다.

## 핵심 한 줄

Cilium 은 노드당 Agent(DaemonSet)가 데이터플레인을 제어하고, 클러스터당 Operator(Deployment)가 클러스터 스코프 작업을 맡으며, CNI 플러그인(호스트 바이너리)이 Pod 네트워크 셋업의 진입점을, Hubble(Observer 내장 + Relay/UI)이 관찰성 평면을 담당하는 네 컴포넌트 토폴로지입니다. BPF 프로그램을 hook 에 부착하고 BPF map 을 갱신하는 주체는 노드 로컬의 Agent 이며, 이것이 이전 세션이 남긴 질문에 대한 직접 답입니다.

## 한 그림으로 보는 클러스터 토폴로지

네 컴포넌트의 배치와 핵심 상호작용 화살표를 한 장의 다이어그램으로 보면 전체 토폴로지가 드러납니다.

```
       ──── 컨트롤 플레인 (k8s API server) ────
              ▲            ▲            ▲
          watch│         CRD│      peer │discovery
         /CRUD │        CRUD│           │
     ┌─────────┘             │           └────────┐
     │                       │                    │
┌────┴──────┐        ┌────────┴────────┐     ┌────┴───────────┐
│ Operator  │        │  Hubble Relay   │     │   Hubble UI    │
│Deployment │        │   Deployment    │     │  Deployment    │
│ (cluster) │        │   (cluster)     │     │(cluster, opt.) │
└───────────┘        └───────▲─────────┘     └────────────────┘
                             │ peer gRPC
┌────────────────────────────┴────────────────────────────────┐
│  노드  (DaemonSet — 모든 노드에 반복)                              │
│                                                             │
│  kubelet ──CNI ADD/DEL──▶ /opt/cni/bin/cilium-cni           │
│                             (veth · IP · 라우트 셋업 후          │
│                              endpoint 등록 위임, Unix socket)  │
│                                  │                          │
│                                  ▼                          │
│               ┌──────────────────────────────────┐          │
│               │   Cilium Agent  (DaemonSet Pod)  │          │
│               │  ├ k8s watch                     │          │
│               │  ├ BPF compile / load / attach   │          │
│               │  ├ BPF map write                 │          │
│               │  └ Hubble Observer (내장)          │          │
│               └──────────────┬───────────────────┘          │
│                              │ bpf() / attach               │
│                              ▼                              │
│                  커널  (XDP / tc / cgroup hook, BPF map)     │
└─────────────────────────────────────────────────────────────┘
```

다이어그램에서 노드 로컬 평면(Agent, CNI, kubelet)과 클러스터 스코프 평면(Operator, Hubble Relay/UI)이 나뉘어 있음을 확인할 수 있습니다. kubelet 은 Cilium 의 존재를 알지 못하고 표준 CNI 프로토콜로 `cilium-cni` 바이너리를 호출할 뿐입니다. Hubble Observer 는 별도 프로세스가 아니라 Agent 프로세스 안에 내장되어 있습니다.

## 메커니즘 — 컴포넌트별 상세

각 컴포넌트를 배포 형태, 스코프, 핵심 책임, 상호작용 주체의 네 축으로 비교합니다.

### Cilium Agent — 노드 로컬 데이터플레인 컨트롤러

Agent 는 DaemonSet 으로 배포됩니다. 노드당 하나의 Pod 가 배치되어 그 노드의 데이터플레인을 전적으로 책임집니다. 코드는 `daemon/` 에 위치합니다.

Agent 의 핵심 책임은 네 가지입니다.

- k8s API server 를 watch 하여 Pod / Service / EndpointSlice / NetworkPolicy 의 변화를 감지합니다. 이전 세션(0005)에서 다룬 Informer 패턴을 재활용합니다.
- eBPF C 소스를 컴파일·로드(`bpf()` 시스템 콜)하고 XDP / tc / cgroup hook 에 attach 합니다. 0008 에서 다룬 세 훅을 실제로 다루는 주체가 바로 Agent 입니다.
- `cilium_lb4_services` / policy map / endpoint map 등 BPF map 을 k8s 상태에 맞게 갱신합니다.
- CNI 플러그인의 요청을 받는 Unix socket 서버(`/var/run/cilium/cilium.sock`)와 Hubble Observer gRPC 서버를 같은 프로세스 안에서 호스팅합니다.

Agent 의 코드 경로 — k8s 이벤트가 어떻게 BPF map write 로 이어지는지 — 는 이후 코드 유닛에서 단일 경로를 따라 추적합니다.

### Cilium Operator — 클러스터 스코프 작업 코디네이터

Operator 는 Deployment 로 배포됩니다. 클러스터당 1~2 개 레플리카에 leader election 을 적용해 단일 액티브 인스턴스를 유지합니다. 코드는 `operator/` 에 위치합니다.

Operator 가 담당하는 작업은 노드당 한 번이 아니라 클러스터 전체를 대상으로 단 한 번만 처리해야 의미 있는 유형입니다. IPAM 클러스터 풀 코디네이션, 사용되지 않는 CiliumIdentity garbage collection, stale CiliumEndpoint 정리, CiliumNetworkPolicy derivative 처리가 대표적입니다. Agent 와의 분업 원칙은 간명합니다. 노드 로컬 데이터플레인 제어는 Agent 가, 클러스터 단일 책임자가 필요한 작업은 Operator 가 처리합니다. Operator 는 커널 BPF 에 직접 손대지 않습니다.

### CNI 플러그인 — kubelet 과 Agent 사이의 어댑터

`cilium-cni` 는 DaemonSet Pod 가 아니라 호스트 파일시스템에 설치되는 바이너리입니다. Agent DaemonSet 의 init container 가 `install-plugin.sh` 를 실행해 `/opt/cni/bin/cilium-cni` 로 복사하고, kubelet 은 Pod 를 생성할 때마다 이 바이너리를 직접 exec 하여 CNI ADD 요청을 전달합니다. 코드는 `plugins/cilium-cni/` 에 위치합니다.

`cilium-cni` 는 두 단계로 동작합니다. 먼저 Pod 의 네트워크 네임스페이스에 진입해 veth 페어를 생성하고, IP 를 할당하고, 라우트를 설정합니다. 그다음 로컬 Agent 의 Unix socket(`/var/run/cilium/cilium.sock`)으로 endpoint 등록과 정책 attach 를 요청하고 종료합니다. 즉 CNI 플러그인은 Pod 의 가상 네트워크 인터페이스를 직접 만들되, 그 위의 Cilium 데이터플레인 제어권은 Agent 에게 넘기는 어댑터입니다. kubelet 은 Cilium 의 존재를 모르고 표준 CNI 프로토콜만 씁니다. 이 인터페이스 분리가 Cilium 을 다른 CNI 구현과 교체 가능하게 만드는 구조적 이유입니다.

### Hubble — 관찰성 평면

Hubble 은 세 부분으로 구성됩니다.

- Observer 는 Agent 프로세스에 내장되어 있습니다(`pkg/hubble/`). BPF datapath 가 내보내는 perf 이벤트와 ringbuf 이벤트를 수신해 flow 스트림으로 변환하고 gRPC 로 노출합니다. Agent 와 같은 프로세스이므로 per-node 로 존재합니다.
- Hubble Relay 는 별도의 Deployment 로 배포됩니다(`hubble-relay/`). 모든 노드의 Observer 에 peer gRPC 로 연결해 클러스터 전체의 flow 를 aggregate 합니다. 따라서 클러스터 단위의 흐름 추적은 Relay 가 있어야 가능합니다.
- 소비자는 Hubble CLI(`hubble/`), Hubble UI(별도 Deployment, 옵션), Prometheus exporter 입니다.

플로우 1 건이 BPF 이벤트에서 Relay 를 거쳐 소비자에게 도달하는 경로의 상세는 이후 Hubble 유닛에서 다룹니다.

## 마무리

### 네 컴포넌트, 네 책임

BPF 프로그램을 hook 에 부착하고 BPF map 을 갱신하는 것은 노드당 한 개씩 배포되는 Agent 입니다. 클러스터 스코프 작업은 단일 leader 로 동작하는 Operator 가 맡고, Agent 는 데이터플레인에만 집중합니다. Pod 생성 시의 네트워크 셋업은 kubelet 이 호출하는 `cilium-cni` 바이너리가 진입점이 되어 Agent 에 제어권을 넘기는 2 단계 구조로 처리됩니다. 관찰성은 Agent 안에 내장된 Hubble Observer 와 클러스터 집계용 Relay 의 조합으로 구성됩니다.

본 유닛이 다루지 않은 부분 — Agent 의 informer 코드 경로, BPF C 프로그램의 패킷 처리, policy map 키 구조, Hubble 의 내부 데이터 경로, identity 시스템 내부, Envoy / Clustermesh / IPAM 내부 — 은 후속 유닛 또는 선택적 확장에서 다룹니다.

### 다음 다리 — Cilium Service LB (ClusterIP)

컴포넌트의 자리가 정해졌으니, 다음 질문은 "그 자리에 놓인 BPF map 이 KUBE-* 체인을 어떻게 대체하는가" 입니다. 다음 세션은 ClusterIP 단일 시나리오에서 `cilium_lb4_services` / `cilium_lb4_backends` map 의 구조를 0006 의 KUBE-SVC / KUBE-SEP 체인과 1:1 로 대응시켜 봅니다.

## 부록 — 4축 비교 요약

| 축 | Agent | Operator | CNI 플러그인 | Hubble |
| --- | --- | --- | --- | --- |
| 배포 형태 | DaemonSet | Deployment (leader election) | 호스트 바이너리 (`/opt/cni/bin/`) | Observer = Agent 내장, Relay/UI = Deployment |
| 스코프 | 노드 로컬 | 클러스터 스코프 | 노드당 (kubelet 이 호출) | Observer = 노드 로컬, Relay = 클러스터 스코프 |
| 핵심 책임 | BPF load/attach, BPF map 갱신, k8s watch | IPAM 코디, identity GC, endpoint GC | Pod veth·IP·라우트 셋업, Agent 에 endpoint 위임 | BPF 이벤트 → flow 스트림 노출·aggregate |
| 상호작용 | k8s API, 커널 BPF, CNI 백엔드, Hubble Observer | k8s API (CRD), (옵션) KVStore | kubelet (CNI), Agent (Unix socket) | Agent peer gRPC, k8s API (peer discovery) |

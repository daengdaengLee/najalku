# kube-proxy: Service 가상 IP를 커널 규칙으로 실체화하기

## ClusterIP 패킷은 어떻게 Pod에 도달하는가 - kube-proxy 역할과 핵심 동작 분석

* **대주제:** k8s 네트워크 스택
* **중주제:** kube-proxy (iptables 모드 중심)

---

## 목차

1. 핵심 한 줄 요약
2. 전체 그림 한눈에 보기
3. 사전 학습 — 전제 개념
4. kube-proxy의 위치와 구조
5. 핵심 동작 루프 — 무엇을 감시하고 무엇을 만드는가
6. 모드(백엔드) 개요
7. ReplicaSet Controller와의 패턴 비교 (0005 → 0006 다리)
8. kube-proxy가 하지 않는 일
9. 다음 세미나 예고

---

## 1. 핵심 한 줄 요약

> **"Service의 가상 IP를 노드 커널 규칙으로 실체화"**

* ClusterIP(Service 의 기본 타입)는 어떤 네트워크 인터페이스에도 바인딩되지 않은 "논리적 가상 IP"
* 이 주소로 향하는 패킷이 실제 Pod에 도달하려면, 누군가 노드 커널에서 목적지를 바꿔치기해 줘야 함
* 그 "누군가"가 kube-proxy (정확히는 바꿔치기 규칙을 생성·동기화하는 역할)

---

## 2. 전체 그림 한눈에 보기

본격 분석 전에 두 관점의 다이어그램으로 kube-proxy의 위치를 먼저 잡고 들어간다.

* **설정 흐름 (Setup)**: kube-proxy가 언제 무엇을 보고 규칙을 심는가
* **패킷 흐름 (Runtime)**: 심어진 규칙을 따라 패킷이 Pod에 도달하는 과정

### 2.1 설정 흐름 — Pod 생성부터 iptables 규칙까지

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Phase 1: Pod 생성 → IP 할당
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

사용자
 │  kubectl apply -f deployment.yaml
 ▼
API Server ───── Deployment 오브젝트 저장
                   │
                   │  (Deployment Controller 감지)
                   ▼
              ReplicaSet 생성
                   │
                   │  (ReplicaSet Controller 감지)  ← 0005 세미나 내용
                   ▼
              Pod 오브젝트 생성 (podIP 없음, nodeName 없음)
                   │
                   │  (Scheduler 감지)
                   ▼
              노드 배정 → Pod.nodeName = "node-1" 기록
                   │
                   │  (node-1의 kubelet 감지)
                   ▼
              kubelet → CRI 런타임에 Pod sandbox 생성 요청
                   │
                   │  (CRI 런타임이 네트워크 네임스페이스 생성 후 CNI 호출)
                   ▼
              [CNI 플러그인]
              Pod 전용 가상 네트워크 인터페이스(veth) 생성
              IP 주소 할당: 10.244.0.5
                   │
                   │  (네트워크 준비 완료 후 kubelet이 앱 컨테이너 시작 요청)
                   ▼
              앱 컨테이너 실행
                   │
                   ▼
API Server ───── Pod.status.podIP = 10.244.0.5 업데이트



━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Phase 2: Service 생성 → EndpointSlice 기록
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

사용자
 │  kubectl apply -f service.yaml
 ▼
API Server ───── Service 오브젝트 저장
                 ClusterIP 자동 부여: 10.96.15.42
                   │
                   │  (EndpointSlice Controller 감지)
                   ▼
              Service selector와 매칭되는 Pod 탐색
              → 10.244.0.5 (readiness는 condition으로 기록)
                   │
                   ▼
API Server ───── EndpointSlice 생성/업데이트
                 내용: [{ip: 10.244.0.5, port: 8080, ready: true}]



━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Phase 3: kube-proxy → iptables 규칙 생성
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[node-1의 kube-proxy]  ← 항상 watch 중
 │
 ├── Service 변경 감지 (OnServiceAdd)
 │    ClusterIP: 10.96.15.42:80
 │
 └── EndpointSlice 변경 감지 (OnEndpointSliceAdd)
      backends: [10.244.0.5:8080]
 │
 ▼  syncProxyRules() 실행
    (iptables-restore 로 규칙 일괄 적용)

[node-1 커널 iptables (nat 테이블)]

  PREROUTING / OUTPUT 체인
    └── KUBE-SERVICES 로 점프

  KUBE-SERVICES 체인
    └── dst == 10.96.15.42:80  →  KUBE-SVC-XXXX로 이동

  KUBE-SVC-XXXX 체인  (부하 분산 — Pod가 여러 개면 확률 분배)
    └── 100%  →  KUBE-SEP-YYYY로 이동   (※ Pod 1개 가정)

  KUBE-SEP-YYYY 체인  (실제 DNAT)
    └── 목적지를 10.244.0.5:8080 으로 바꿈
```

### 2.2 패킷 흐름 — 외부/내부 → Pod

패킷이 처음 만나는 체인이 다를 뿐, 이후 흐름은 같다. 두 케이스를 나란히 본다.

#### Case A: 클러스터 내부 → ClusterIP

```
Pod A (10.244.0.2, 같은 클러스터 안)
 │  curl http://10.96.15.42:80
 │
 │  [패킷 출발]
 │  src: 10.244.0.2    dst: 10.96.15.42:80
 ▼
커널 OUTPUT 훅  ← "내가 보내는 패킷"이 커널을 통과하는 시점
 │
 ▼  KUBE-SERVICES 체인 검사
    "dst가 10.96.15.42:80 이다 → KUBE-SVC-XXXX"
 │
 ▼  KUBE-SVC-XXXX 체인
    "어느 Pod로 보낼까? → KUBE-SEP-YYYY"
 │
 ▼  KUBE-SEP-YYYY 체인
    DNAT: dst 10.96.15.42:80  →  10.244.0.5:8080
 │
 │  [패킷 변환 후]
 │  src: 10.244.0.2    dst: 10.244.0.5:8080  ← 목적지 바뀜
 ▼
Pod B (10.244.0.5) 수신 → 처리 → 응답
 │
 ▼  커널 Conntrack(연결 추적)이 역방향 자동 복원
    응답 패킷: src 10.244.0.5:8080  →  10.96.15.42:80 으로 복원
 │
 ▼
Pod A 수신 — ClusterIP에서 응답이 온 것처럼 보임
```

#### Case B: 클러스터 외부 → NodePort

```
외부 클라이언트 (인터넷)
 │  curl http://10.0.0.1:30080   ← 노드의 실제 IP:NodePort
 │
 │  [패킷 도착]
 │  src: 203.0.113.1    dst: 10.0.0.1:30080
 ▼
node-1 (IP: 10.0.0.1) 수신
 │
 ▼  커널 PREROUTING 훅  ← "외부에서 들어온 패킷"이 처음 만나는 지점
    KUBE-SERVICES 체인 → KUBE-NODEPORTS 체인 검사
    "dst port가 30080 이다 → KUBE-SVC-XXXX"
 │
 ▼  KUBE-SVC-XXXX 체인
    "어느 Pod로? → KUBE-SEP-YYYY"
 │
 ▼  KUBE-SEP-YYYY 체인
    DNAT: dst 10.0.0.1:30080  →  10.244.0.5:8080
 │
 │  [패킷 변환 후]
 │  src: 203.0.113.1    dst: 10.244.0.5:8080
 ▼
Pod B (10.244.0.5) 수신 → 응답
```

#### 두 케이스 합류 구조

```
진입점
 │
 ├── Case A (내부): OUTPUT      ─┐
 └── Case B (외부): PREROUTING  ─┘
                                 │
                                 ▼
                          KUBE-SERVICES
                       (공통 진입 — ClusterIP면 직행,
                        NodePort면 KUBE-NODEPORTS 경유)
                                 │
                                 ▼
                          KUBE-SVC-XXXX
                           (부하 분산)
                                 │
                                 ▼
                          KUBE-SEP-YYYY
                           (DNAT 실행)
                                 │
                                 ▼
                              Pod 도달
```

### 2.3 핵심 인사이트

* 설정 흐름(§2.1)과 실행 흐름(§2.2)은 **완전히 분리**됨
* kube-proxy는 §2.1 Phase 3에서 한 번 규칙을 심으면 트래픽 경로에서 빠짐 → 모든 패킷은 **커널이 직접** 처리
* 이 성질이 iptables 모드의 빠른 데이터플레인의 근거 — 그러나 동시에 한계의 근원이기도 함(아래에서 상세 설명)
* 이후 각 단계를 자세히 분석

---

## 3. 사전 학습 — 전제 개념

### 3.1 Service 추상

kube-proxy 관점에서 Service는 "어떤 포트로 들어온 트래픽을 어떤 Pod 집합으로 분산할지"를 선언하는 명세다.

| Service 타입 | kube-proxy 관점 |
|-------------|----------------|
| ClusterIP | 클러스터 내부에서만 유효한 가상 IP — 노드마다 iptables DNAT 규칙 생성 |
| NodePort | ClusterIP에 더해, 모든 노드의 특정 포트도 진입점으로 추가 |
| LoadBalancer | NodePort에 더해, 외부 LB가 붙여준 IP도 진입점으로 추가 |
| Headless (`clusterIP: None`) | 가상 IP 없음 → kube-proxy 규칙 **미생성**, DNS 전담 |
| ExternalName | CNAME 리다이렉트 — kube-proxy 규칙 **미생성** |

* **핵심**: ClusterIP는 아무 인터페이스에도 없는 주소 → 패킷이 커널을 통과할 때 NAT으로 목적지를 바꿔치기해야만 도달 가능
* 이 역할이 kube-proxy(iptables 모드)에서는 **DNAT 규칙**으로 구현됨

### 3.2 EndpointSlice

* Pod IP·포트의 실제 목록을 담은 API 오브젝트 (k8s 1.21+ 기본)
* 각 endpoint에는 `ready`/`serving`/`terminating` **condition이 표기**되며, ready 여부로 항목을 빼지 않고 모두 포함시킨 뒤 condition으로 구분
* **EndpointSlice Controller**(`pkg/controller/endpointslice/`)가 생성·유지
* kube-proxy는 이 오브젝트를 **소비**해 DNAT 목적지 후보 목록을 만들 뿐 (condition을 보고 후보 여부 결정)

```
[API Server]
  Service      ───► EndpointSlice Controller ──► EndpointSlice
  (ClusterIP:port, selector)                    (pod IP:port 목록)
                                                   │
                                           kube-proxy가 읽어서
                                           iptables 규칙으로 변환
```

### 3.3 Linux netfilter / DNAT

* 리눅스 커널의 **netfilter** 프레임워크 위에 **iptables** 규칙이 동작
* `nat` 테이블의 `PREROUTING` / `OUTPUT` 훅에서 목적지 IP를 바꾸는 것이 **DNAT**
* kube-proxy는 `iptables-restore` 명령으로 이 규칙 셋을 커널에 일괄 적용

```
패킷 (dest: ClusterIP:80)
  │
  ▼ PREROUTING / OUTPUT (nat table)
iptables DNAT 규칙 매칭
  │
  ▼
패킷 (dest: PodIP:8080)  ──► Pod
```

### 3.4 Informer + Reconcile 패턴 (0005 복습)

* 0005에서 배운 `SharedIndexInformer` → `EventHandler` → `WorkQueue` → `Reconcile` 패턴이 kube-proxy에서도 동일하게 쓰임
* **차이**: kube-proxy는 per-key WorkQueue 대신 `BoundedFrequencyRunner` 사용 → 섹션 7에서 자세히

---

## 4. kube-proxy의 위치와 구조

### 4.1 노드별 DaemonSet

* `kube-system` 네임스페이스에 DaemonSet으로 배포 → **모든 노드에 1개 Pod**
* 컨트롤 플레인 컴포넌트가 아님 — 각 노드의 **데이터 플레인** 측에서 동작
* 프로세스 이름: `kube-proxy`

```
[클러스터]

  [컨트롤 플레인]              [워커 노드 A]           [워커 노드 B]
  API Server                  kube-proxy Pod           kube-proxy Pod
     │                            │  watch                   │  watch
     │◄──────────────────────────►│  sync rules              │  sync rules
     │◄──────────────────────────────────────────────────────►│
     │
  EndpointSlice Controller ──► EndpointSlice (API Server에 저장)
```

* 각 노드의 kube-proxy는 **자기 노드**의 iptables 규칙만 관리
* 노드 간 kube-proxy는 서로 통신하지 않음 — 모두 API Server를 통해 동일 상태 구독

### 4.2 전체 동작 구조

```
[Node]

  kube-proxy
    │
    ├── watch ──► API Server
    │               ├── Service
    │               ├── EndpointSlice
    │               ├── Node
    │               └── ServiceCIDR
    │
    ▼ 변경 감지 시 (BoundedFrequencyRunner 코얼레싱)
    │
    └── syncProxyRules()
          │
          ▼
    Linux kernel (iptables nat 테이블)
    KUBE-SERVICES → KUBE-SVC-XXX → KUBE-SEP-XXX → DNAT
          │
          ▼
    Pod 트래픽 도달
```

### 4.3 진입점

> `cmd/kube-proxy/proxy.go`

```go
func main() {
    command := app.NewProxyCommand()
    code := cli.Run(command)
    os.Exit(code)
}
```

* `NewProxyCommand()` → Cobra RunE → `opts.Run(ctx)` → `runLoop()`
* `runLoop()` 안에서 Informer 연결 + `go s.Proxier.SyncLoop()` 시작
  * `cmd/kube-proxy/app/server.go:595–627`

---

## 5. 핵심 동작 루프 — 무엇을 감시하고 무엇을 만드는가

### 5.1 감시 대상

> `cmd/kube-proxy/app/server.go:577-622`

```
serviceInformerFactory       ──► Service (Headless 제외 필터)
endpointSliceInformerFactory ──► EndpointSlice (Headless 제외 필터)
informerFactory              ──► ServiceCIDR (MultiCIDRServiceAllocator feature gate)
s.NodeManager.NodeInformer() ──► Node (노드 레이블·토폴로지)
```

* Service와 EndpointSlice는 **별도의 InformerFactory**로 생성 — Headless 항목을 LabelSelector/FieldSelector로 필터링해 제외
  * `spec.clusterIP != None` (Service), `!service.kubernetes.io/headless` (EndpointSlice)
* kube-proxy가 읽는 상태는 **API Server에 저장된 것** — 직접 DB를 보지 않음

### 5.2 핸들러 인터페이스

> `pkg/proxy/config/config.go:40, 57`

```go
type ServiceHandler interface {       // L40
    OnServiceAdd(service *v1.Service)
    OnServiceUpdate(oldService, service *v1.Service)
    OnServiceDelete(service *v1.Service)
    OnServiceSynced()
}

type EndpointSliceHandler interface {  // L57
    OnEndpointSliceAdd(endpointSlice *discoveryv1.EndpointSlice)
    OnEndpointSliceUpdate(oldEndpointSlice, newEndpointSlice *discoveryv1.EndpointSlice)
    OnEndpointSliceDelete(endpointSlice *discoveryv1.EndpointSlice)
    OnEndpointSlicesSynced()
}
```

* iptables `Proxier`가 이 인터페이스를 구현 → `RegisterEventHandler`로 등록
* 콜백 호출 시점: Informer EventHandler가 변경 이벤트를 받을 때

### 5.3 변경 → 규칙 동기화 흐름

```
Informer 이벤트
  │  (Service Add/Update/Delete, EndpointSlice Add/Update/Delete)
  ▼
OnServiceUpdate() / OnEndpointSliceUpdate()  (pkg/proxy/iptables/proxier.go:467~)
  │  변경 내용을 로컬 트래커에 누적
  │  servicechangetracker.go / endpointschangetracker.go
  ▼
proxier.Sync()                               (proxier.go:426)
  │  syncRunner.Run() 호출
  ▼
BoundedFrequencyRunner
  │  minSyncPeriod ~ FullSyncPeriod 범위 안에서 다수 이벤트를 묶어 1회 실행
  ▼
syncProxyRules()                             (proxier.go:638)
  │  현재 Service·EndpointSlice 전체 상태로 iptables 규칙 재생성
  │  iptables-restore 로 커널에 일괄 적용
  ▼
Linux kernel iptables 규칙 갱신 완료
```

**핵심 설계 포인트 — 전체 상태 풀 동기화**

* kube-proxy는 "이 Service가 바뀌었으니 이 규칙만 수정"하지 않음
* 항상 **전체 노드 iptables 규칙을 처음부터 재생성**해 일괄 적용
* 이유: 한 Service 변경이 KUBE-SERVICES, KUBE-SVC-*, KUBE-SEP-* 등 여러 체인에 연쇄적으로 영향 → 부분 수정보다 전체 재생성이 안전하고 단순

### 5.4 BoundedFrequencyRunner — 빈도 제한

* 이벤트가 몰려도 `syncProxyRules`를 무한 반복하지 않도록 빈도 제어
* `minSyncPeriod`: 두 번의 syncProxyRules 사이 최소 대기 시간 (기본 0 — 즉시 반응)
* `FullSyncPeriod`(`--iptables-sync-period`): 변경이 없어도 강제 전체 재동기화 주기 (기본 30초)
* `Sync()` 호출 → `syncRunner.Run()` → 러너가 타이머 관리, 실제 실행은 `syncProxyRules()`

---

## 6. 모드(백엔드) 개요

### 6.1 사용 가능한 모드

> `cmd/kube-proxy/app/options.go:130`

```
--proxy-mode: 'iptables' (Linux 기본) | 'ipvs' | 'nftables' | 'kernelspace' (Windows 전용)
```

> `cmd/kube-proxy/app/server_linux.go:47-49` — 모드 기본값 결정

```go
if config.Mode == "" {
    config.Mode = proxyconfigapi.ProxyModeIPTables  // 빈 값이면 iptables
}
```

| 모드 | 플랫폼 | 특성 |
|------|--------|-----|
| `iptables` | Linux (기본) | netfilter nat 테이블에 체인 방식으로 규칙 작성; 본 세미나 대상 |
| `ipvs` | Linux | 커널 IPVS(L4 LB)로 부하 분산; **현재 deprecated** — nftables 권장 |
| `nftables` | Linux | iptables 후속 규칙 체계; 더 효율적인 집합(set) 기반 매칭 |
| `kernelspace` | Windows | Windows HNS(Host Networking Service) 기반 |

> **ipvs deprecated 경고** `server_linux.go:187-188`
> ```
> "The ipvs proxier is now deprecated and may be removed in a future release. Please use 'nftables' instead."
> ```
> → iptables도 장기적으로 nftables 또는 Cilium(eBPF) 방향 — 섹션 9 예고 참조

### 6.2 Linux 모드 디스패치

> `cmd/kube-proxy/app/server_linux.go`

```
createProxier()
  ├── L128: ProxyModeIPTables  → iptables.NewProxier()     (또는 DualStack)
  ├── L178: ProxyModeIPVS      → ipvs.NewProxier()         (deprecated 경고)
  └── L245: ProxyModeNFTables  → nftables.NewProxier()     (또는 DualStack)
```

### 6.3 본 세미나 범위

* **iptables 모드만 상세 분석** (다음 라운드: `syncProxyRules()` 코드 워크스루)
* 다른 모드는 위 비교 표로 대체

---

## 7. ReplicaSet Controller와의 패턴 비교 (0005 → 0006 다리)

### 7.1 공통점 — Informer 기반 Reconcile

kube-proxy도 0005와 동일하게 client-go `SharedIndexInformer`를 사용해 API 변경을 감지하고 원하는 상태(iptables 규칙)로 수렴한다.

```go
// pkg/proxy/config/config.go:85
endpointSliceInformer.Informer().AddEventHandlerWithResyncPeriod(
    cache.ResourceEventHandlerFuncs{
        AddFunc:    result.handleAddEndpointSlice,
        UpdateFunc: result.handleUpdateEndpointSlice,
        DeleteFunc: result.handleDeleteEndpointSlice,
    },
    resyncPeriod,
)
```

### 7.2 차이점 — Reconcile 단위와 큐 방식

|  | ReplicaSet Controller | kube-proxy |
|--|----------------------|------------|
| **감시 대상** | ReplicaSet, Pod | Service, EndpointSlice, Node, ServiceCIDR |
| **큐 방식** | per-key WorkQueue | 없음 → `BoundedFrequencyRunner` |
| **Reconcile 단위** | 키(`namespace/name`) 1개 | 노드 전체 iptables 규칙 재생성 |
| **트리거 방식** | `processNextWorkItem` 루프 | `OnXxxUpdate` 콜백 → `Sync()` |
| **빈도 제어** | rate-limited 재시도 backoff | `minSyncPeriod` ~ `FullSyncPeriod` |
| **Reconcile 함수** | `syncReplicaSet(key)` | `syncProxyRules()` (키 인자 없음) |

### 7.3 왜 kube-proxy는 per-key WorkQueue를 쓰지 않는가

ReplicaSet Controller는 `namespace/name` 키 하나로 특정 ReplicaSet의 파드 수만 조정하면 된다. 변경 대상이 명확하게 분리된다.

kube-proxy는 그렇지 않다. Service 하나가 바뀌면:
- KUBE-SERVICES 체인의 해당 엔트리를 수정해야 하고
- KUBE-SVC-XXXX 체인(부하 분산 확률 재계산)도 바꿔야 하며
- KUBE-SEP-XXXX 체인(실제 DNAT 목적지)도 함께 바꿔야 한다

이 체인들은 서로 연결되어 있고 `iptables-restore`는 **파일 단위로 일괄 적용**하는 방식이라 부분 수정보다 전체 재생성이 더 단순하고 안전하다. 따라서 "어떤 키가 바뀌었는가"를 추적하는 WorkQueue보다 "지금 전체 상태가 무엇인가"를 기준으로 풀 동기화하는 BoundedFrequencyRunner가 더 자연스럽다.

---

## 8. kube-proxy가 하지 않는 일

| 기능 | 실제 담당 컴포넌트 |
|------|-----------------|
| Pod에 IP 주소 할당 | **CNI 플러그인** (e.g. Calico, Flannel) |
| Pod 네트워크 인터페이스(veth) 생성·설정 | **CNI 플러그인** |
| EndpointSlice 생성·업데이트 | **EndpointSlice Controller** (`pkg/controller/endpointslice/`) |
| L7 HTTP 라우팅 (경로, 헤더 기반) | **Ingress Controller / Gateway Controller** |
| NetworkPolicy 시행 (L3/L4 접근 제어) | **CNI 플러그인** (e.g. Calico, Cilium) |
| LoadBalancer 외부 IP 프로비저닝 | **cloud-controller-manager** |
| Headless Service · ExternalName Service 처리 | **CoreDNS** |
| Service status(`.status.loadBalancer`) 갱신 | **cloud-controller-manager** |

* Headless Service는 kube-proxy Informer 단계에서 **필터링되어 아예 수신되지 않음**
  * serviceInformerFactory 생성 시 `spec.clusterIP != None` FieldSelector 적용 (`server.go:593`)

> 백업

| 기능                                                             | 담당                           | 설명                                                                                                                                                                          |
|----------------------------------------------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ClusterIP·NodePort·LoadBalancer·ExternalIP → Pod IP DNAT 규칙 생성 | **kube-proxy**               | 가상 IP로 들어온 패킷의 목적지를 실제 Pod IP:port로 바꾸는 iptables DNAT 규칙 생성                                                                                                                 |
| 노드 전체 iptables 규칙 일괄 재생성                                       | **kube-proxy**               | Service·EndpointSlice 변경을 watch해 트리거됨. Service 하나가 바뀌어도 KUBE-SERVICES, KUBE-SVC-\*, KUBE-SEP-\* 체인이 연쇄 영향을 받으므로, 부분 수정 대신 항상 전체 상태를 기준으로 규칙을 재구성해 `iptables-restore`로 일괄 적용 |
| Headless Service 스킵                                            | **kube-proxy** (규칙 미생성)      | `clusterIP: None`이라 가상 IP 없음 → DNAT 규칙 불필요. Service Informer의 FieldSelector(`spec.clusterIP!=None`)로 **수신 단계에서 제외**                                                         |
| ExternalName Service 스킵                                        | **kube-proxy** (규칙 미생성)      | DNS CNAME 리다이렉트라 가상 IP 없음. FieldSelector로는 걸러지지 않고 **kube-proxy 내부에서 Service 타입을 보고 무시** (DNS는 CoreDNS가 처리)                                                                 |
| Pod IP 할당·네트워크 인터페이스 생성                                        | CNI 플러그인                     | kubelet이 CRI 런타임을 통해 sandbox를 만들면, **CRI 런타임이 CNI 플러그인을 호출**해 Pod 네트워크 네임스페이스에 인터페이스(예: veth)와 IP를 구성. kube-proxy는 이미 할당된 IP를 EndpointSlice에서 읽어 쓸 뿐                        |
| EndpointSlice 생성·유지                                            | EndpointSlice Controller     | Service selector에 매칭되는 Pod들을 endpoint로 등록하되, 각 endpoint에 `ready`/`serving`/`terminating` **condition을 표기**. kube-proxy는 이를 소비해 `ready=true`인 endpoint만 부하 분산 후보로 사용         |
| L7 HTTP 라우팅                                                    | Ingress / Gateway Controller | 컨트롤러가 L7 라우팅 규칙을 데이터플레인 프록시에 구성하고, **실제 라우팅(경로·헤더·호스트)은 데이터플레인 프록시(Envoy, NGINX 등)가 수행**. kube-proxy는 IP·포트 수준(L3/L4)만 처리                                                   |
| NetworkPolicy 시행                                               | CNI 플러그인                     | Pod 간 접근 제어 규칙 시행. Calico·Cilium 등이 별도 커널 규칙(iptables 또는 eBPF)으로 처리                                                                                                         |
| DNS 해석                                                         | CoreDNS                      | `my-svc.default.svc.cluster.local` → ClusterIP 변환. Headless Service는 **EndpointSlice를 참조해 Pod IP들을 A/AAAA 레코드로 직접 반환** (named port는 SRV로 제공)                                |

---

## 9. 다음 세미나 예고 — `syncProxyRules()` 코드 워크스루

이번 세미나는 kube-proxy의 **역할과 핵심 동작**까지만 다뤘다. 다음 단계:

### 9.1 2라운드 목표

> `pkg/proxy/iptables/proxier.go:638` — `syncProxyRules()` 시작점

1. `syncProxyRules()` 내부 — 5단계 체인 생성 흐름
    ```
    ① 현재 Service·EndpointSlice 상태 수집
    ② KUBE-SERVICES 체인 구성 (ClusterIP:port 별 엔트리)
    ③ KUBE-SVC-XXXX 체인 (Service별 부하 분산 확률)
    ④ KUBE-SEP-XXXX 체인 (Endpoint별 DNAT 목적지)
    ⑤ iptables-restore 로 커널에 일괄 적용
    ```
2. 한 패킷 추적 — `curl ClusterIP:80`에서 Pod에 도달까지
    ```
    KUBE-SERVICES → KUBE-SVC-XXX → KUBE-SEP-XXX → DNAT → Pod
    ```
3. (선택) 미니큐브 데모
    ```bash
    # Service 생성 전/후 iptables 규칙 비교
    minikube ssh -- sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers
    ```

### 9.2 kube-proxy iptables의 한계 — Cilium으로의 다리

* Service 수 증가 → iptables 체인 수 O(n) 증가 → 동기화 지연
* iptables는 순차 매칭 → 규모 커질수록 성능 저하
* 체인 디버깅 어려움
* → **Cilium(eBPF)** 시리즈(0007+)에서 kube-proxy 대체 방식 탐구 예정

---

## 부록: 주요 코드 위치

| 항목 | 경로 |
|------|------|
| 패키지 doc | `pkg/proxy/doc.go:17` |
| 진입점 | `cmd/kube-proxy/proxy.go` → `cmd/kube-proxy/app/server.go` |
| Informer 연결 | `cmd/kube-proxy/app/server.go:577-622` |
| 모드 플래그 | `cmd/kube-proxy/app/options.go:130` |
| Linux 모드 기본값 | `cmd/kube-proxy/app/server_linux.go:47-49` |
| Linux 모드 디스패치 | `cmd/kube-proxy/app/server_linux.go:128, 178, 245` |
| 핸들러 인터페이스 | `pkg/proxy/config/config.go:40 (ServiceHandler), 57 (EndpointSliceHandler)` |
| Informer 등록 | `pkg/proxy/config/config.go:80-97 (NewEndpointSliceConfig)` |
| iptables Sync() | `pkg/proxy/iptables/proxier.go:426` |
| iptables SyncLoop() | `pkg/proxy/iptables/proxier.go:435` |
| iptables syncProxyRules() | `pkg/proxy/iptables/proxier.go:638` |
| 변경 트래커 | `pkg/proxy/servicechangetracker.go`, `pkg/proxy/endpointschangetracker.go` |
| Headless 필터 | `cmd/kube-proxy/app/server.go:590-594` (FieldSelector) |

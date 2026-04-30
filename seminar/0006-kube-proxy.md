# kube-proxy: Service 가상 IP를 커널 규칙으로 실체화하기

## ClusterIP 패킷은 어떻게 Pod에 도달하는가 - kube-proxy 역할과 핵심 동작 분석

* **대주제:** k8s 네트워크 스택
* **중주제:** kube-proxy (iptables 모드 중심)

---

## 목차

1. 핵심 한 줄 요약
2. 전체 그림 한눈에 보기
3. 사전 학습 — 전제 개념
4. reconcile 루프 — 이벤트에서 iptables 규칙까지
5. iptables 모드의 한계와 다음 다리

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
 전제: kube-proxy 배포 위상
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[컨트롤 플레인]              [워커 노드 A]           [워커 노드 B]
 API Server                  kube-proxy Pod           kube-proxy Pod
     ▲                            │ watch                   │ watch
     ├────────────────────────────┘                         │
     └──────────────────────────────────────────────────────┘
     │
 EndpointSlice Controller ──► EndpointSlice 저장
                         (kube-proxy 가 동일하게 watch)

 • DaemonSet — 모든 노드에 1 Pod (kube-system 네임스페이스)
 • 컨트롤 플레인 ✗  데이터 플레인 ✓  자기 노드 iptables 만 관리
 • 노드 간 kube-proxy 직접 통신 없음 — API Server 경유 동기화



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



━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Phase 3.1: kube-proxy 내부 동작 한눈에
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

kube-proxy 프로세스 (각 노드에서 실행 중)
 │
 ├── watch ──► API Server
 │               ├─ Service
 │               ├─ EndpointSlice
 │               ├─ Node
 │               └─ ServiceCIDR
 │
 ▼  변경 감지 시
    BoundedFrequencyRunner 가 짧은 시간 내 변경 묶어 코얼레싱
 │
 └── syncProxyRules() 실행
      └── iptables-restore 로 규칙 일괄 적용
           (앞 Phase 3 의 KUBE-* 체인 갱신)
```

### 2.2 패킷 흐름 — 외부/내부 → Pod

패킷이 처음 만나는 체인이 다를 뿐, 이후 흐름은 같다. 두 케이스를 나란히 본다.

* Pod 발신(Case A) → veth 통해 호스트 진입 → PREROUTING
* 외부 도착(Case B) → 노드 NIC 수신 → PREROUTING
* 호스트 프로세스(kubelet 등) 발신 → OUTPUT
* 어느 진입 훅이든 이후 동일 KUBE-SERVICES 로 합류

#### Case A: 클러스터 내부 → ClusterIP

```
Pod A (10.244.0.2, 같은 클러스터 안)
 │  curl http://10.96.15.42:80
 │
 │  [패킷 출발]
 │  src: 10.244.0.2    dst: 10.96.15.42:80
 ▼
커널 PREROUTING 훅  ← Pod에서 나온 패킷이 host 네트워크로 들어오는 시점
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
 ▼  커널 PREROUTING 훅  ← 외부에서 들어온 패킷이 처음 만나는 지점
    KUBE-SERVICES 체인 → KUBE-NODEPORTS 체인 검사
    "dst port가 30080 이다 → KUBE-SVC-XXXX"
 │
 ▼  KUBE-SVC-XXXX 체인
    "어느 Pod로? → KUBE-SEP-YYYY"
 │
 ▼  KUBE-SEP-YYYY 체인
    DNAT: dst 10.0.0.1:30080  →  10.244.0.5:8080
 │
 ▼  커널 POSTROUTING 훅 → KUBE-POSTROUTING
    MASQUERADE: src 203.0.113.1  →  10.0.0.1
    (NodePort 진입 경로에서 KUBE-MARK-MASQ 로 미리 마킹된 패킷에 SNAT
     — externalTrafficPolicy=Cluster 기본)
 │
 │  [패킷 변환 후]
 │  src: 10.0.0.1    dst: 10.244.0.5:8080   ← 클라이언트 IP 손실(기본 정책)
 ▼
Pod B (10.244.0.5) 수신 → 처리 → 응답
 │
 ▼  커널 Conntrack 역방향 자동 복원
    응답 src: 10.244.0.5:8080  →  10.0.0.1:30080 (un-DNAT)
    응답 dst: 10.0.0.1         →  203.0.113.1   (un-SNAT)
 │
 ▼
외부 클라이언트 수신 — NodePort에서 응답이 온 것처럼 보임
```

**MASQUERADE 이유**

* SNAT 없이 src=클라이언트IP 그대로면: Pod 응답이 자기 노드에서 직접 외부 송출 → node-1 의 conntrack 매칭 실패(비대칭 라우팅) → 연결 깨짐
* src=node-1 IP 로 SNAT → 응답이 node-1 로 복귀 → conntrack 이 un-DNAT/un-SNAT 자동 복원
* `externalTrafficPolicy=Local`: Pod 있는 노드로만 진입 보장 → 대칭 경로 → SNAT 불필요·클라이언트 IP 보존(트레이드오프: Pod 없는 노드 트래픽 드롭)

#### 두 케이스 합류 구조

```
진입점
 │
 ├── Case A (내부, Pod): PREROUTING   ─┐
 └── Case B (외부):      PREROUTING   ─┘
                                       │
                                       ▼
                                KUBE-SERVICES
                                 ├── ClusterIP 매칭
                                 │      → KUBE-SVC-XXXX
                                 └── NodePort 매칭
                                        → KUBE-NODEPORTS
                                           → KUBE-SVC-XXXX
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
* 노드별 kube-proxy 는 자기 노드 규칙만 관리 — 노드 간 직접 통신 없이 API Server 가 유일한 동기화 허브
* kube-proxy는 iptables에 한 번 규칙을 심으면 트래픽 경로에서 빠짐 → 모든 패킷은 **커널이 직접** 처리
* 단 iptables 규칙 매칭은 선형 비용(O(n)) — 대규모 Service/Endpoint 에서는 한계(IPVS·eBPF 모드 대비, 뒤쪽 섹션에서 상세)
* 이후 각 단계를 자세히 분석

---

## 3. 사전 학습 — 전제 개념

### 3.1 Service 추상

kube-proxy 관점에서 Service는 "어떤 포트로 들어온 트래픽을 어떤 Pod 집합으로 분산할지"를 선언하는 명세다.

* **ClusterIP**: 클러스터 내부에서만 유효한 가상 IP — 노드마다 iptables DNAT 규칙 생성
* **NodePort**: ClusterIP에 더해, 모든 노드의 특정 포트도 진입점으로 추가
* **LoadBalancer**: NodePort에 더해, 외부 LB가 붙여준 IP도 진입점으로 추가
* **Headless (`clusterIP: None`)**: 가상 IP 없음 → kube-proxy 규칙 **미생성**, DNS가 Pod IP 목록을 직접 반환
  * 클라이언트가 특정 Pod를 골라야 할 때 (예: StatefulSet)
  * 클라이언트가 직접 부하 분산할 때 (예: gRPC long-lived connection)
* **ExternalName**: CNAME 리다이렉트 — kube-proxy 규칙 **미생성**
  * 외부 DNS를 클러스터 내부 이름으로 포장 — 외부 서비스를 내부 서비스처럼 사용

**핵심**

* ClusterIP는 아무 인터페이스에도 없는 주소 → 패킷이 커널을 통과할 때 NAT으로 목적지를 바꿔치기해야만 도달 가능
* 이 역할이 kube-proxy(iptables 모드)에서는 **DNAT 규칙**으로 구현됨
* Headless·ExternalName 은 kube-proxy 가 관여하지 않는 경계 표시 — 이후 분석은 규칙이 생성되는 ClusterIP/NodePort/LoadBalancer 로 한정

### 3.2 EndpointSlice

* Pod IP·포트의 실제 목록을 담은 API 오브젝트 (k8s 1.21+ GA·기본)
* 각 endpoint에 `ready` / `serving` / `terminating` **condition 표기** — ready가 아니어도 항목을 빼지 않고 모두 포함시킨 뒤 condition으로 구분
  * 종료 중(terminating)인 Pod도 슬라이스에 남음 → 그레이스풀 종료·트래픽 드레인 정책을 condition만 보고 결정 가능
* **EndpointSlice Controller** (`pkg/controller/endpointslice/`, kube-controller-manager 내부) 가 Service·Pod를 watch해 생성·유지
* kube-proxy 입장에선 **읽기 전용 인터페이스** — condition을 보고 DNAT 후보 여부 결정

```
[API Server]
 Service (selector) + Pod (status)
        │
        │ (EndpointSlice Controller 가 변경 감지)
        ▼
EndpointSlice Controller
(kube-controller-manager 내부 컴포넌트)
        │
        │ EndpointSlice 생성·유지
        ▼
[API Server]
 EndpointSlice 저장
  • pod IP:port 목록
  • condition: ready / serving / terminating
        │
        │ (kube-proxy 가 변경 감지)
        ▼
kube-proxy
(각 노드 — condition 보고 DNAT 후보 결정 → iptables 규칙으로 변환)
```

**핵심**

* EndpointSlice는 producer(Controller) ↔ consumer(kube-proxy) 사이의 **계약**
* Controller는 "Pod가 살아있고 트래픽 받을 수 있는가?" 를 condition에 기록
* kube-proxy는 그 condition만 보고 iptables DNAT 후보 결정 — Pod 오브젝트는 직접 watch하지 않음

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

**핵심**

* 두 훅 의미: `PREROUTING`은 **네트워크 인터페이스로 수신된** 패킷이 라우팅되기 전, `OUTPUT`은 **호스트 네트워크 네임스페이스의 프로세스**(kubelet, kube-proxy 등)가 직접 생성한 패킷. 일반 Pod 트래픽은 자체 네임스페이스의 veth 로 나가 PREROUTING 경로로 들어옴. 어느 경로든 동일한 KUBE-* 규칙 셋을 거침
* 응답 패킷의 역방향 복원(un-DNAT)은 커널 **Conntrack**(연결 추적)이 자동 처리 — kube-proxy 규칙은 단방향만 정의하면 됨
* kube-proxy는 `iptables-restore`로 규칙 셋만 동기화 — 실제 패킷 변환은 **커널 netfilter가 수행**

### 3.4 Informer + Reconcile 패턴 (0005 복습)

* 0005에서 배운 `SharedIndexInformer` → `EventHandler` → `WorkQueue` → `Reconcile` 패턴이 kube-proxy에서도 동일하게 쓰임
* **차이**: kube-proxy는 per-key WorkQueue 대신 `BoundedFrequencyRunner` 사용 → 뒤 절에서 자세히

---

## 4. reconcile 루프 — 이벤트에서 iptables 규칙까지

앞 절에서 예고한 Informer + Reconcile 패턴의 실제 구현체. 세 흐름을 순서대로 분해한다.

* **§4.1** kube-proxy 가 시작될 때 watch 등록과 핸들러 결선이 어떻게 이루어지는가
* **§4.2** Informer 이벤트가 올라왔을 때 `syncProxyRules` 호출에 이르기까지의 호출 체인
* **§4.3** `syncProxyRules` 내부에서 어떤 계산이 일어나 iptables 규칙이 갱신되는가
* **§4.4** `BoundedFrequencyRunner` 가 이벤트 폭주를 어떻게 제어하는가

### 4.1 watch 등록과 핸들러 결선

`main()` 진입부터 Informer 결선·핸들러 등록·SyncLoop 기동까지 한 트리.

```
main()                                              cmd/kube-proxy/proxy.go:29
  └─ app.NewProxyCommand()                          cmd/kube-proxy/app/server.go:98
      └─ Cobra RunE                                 server.go:110
          └─ opts.Run(ctx)                          options.go:361   (호출 지점 server.go:133)
              ├─ newProxyServer(...)                server.go:181    (Proxier·NodeManager 등 구성)
              └─ opts.runLoop(ctx)                  options.go:388
                  └─ go o.proxyServer.Run(ctx)      options.go:395
                      └─ (s *ProxyServer).Run       server.go:526
                          ├─ Headless/proxy-name 필터용 LabelSelector 준비
                          │     ├─ noHeadlessEndpoints                 server.go:568
                          │     └─ noProxyName                         server.go:563
                          ├─ endpointSliceInformerFactory 생성         server.go:580
                          │    └─ LabelSelector !service.kubernetes.io/headless
                          │                                            server.go:582
                          ├─ serviceInformerFactory 생성               server.go:590
                          │    ├─ LabelSelector !service.kubernetes.io/service-proxy-name
                          │    │                                       server.go:592
                          │    └─ FieldSelector spec.clusterIP != None server.go:593
                          ├─ NewServiceConfig + RegisterEventHandler(s.Proxier)
                          │                                            server.go:595, 596
                          ├─ NewEndpointSliceConfig + RegisterEventHandler(s.Proxier)
                          │                                            server.go:599, 600
                          └─ go s.Proxier.SyncLoop()                   server.go:627
```

**핵심**

* `(*ProxyServer).Run` 안에서 두 가지가 결선됨:
  1. Informer factory + Config + `RegisterEventHandler` 로 **이벤트 수신 경로 구축**
  2. `go s.Proxier.SyncLoop()` 로 **소비 루프 기동**
* 두 InformerFactory 는 TweakListOptions 로 Headless·proxy-name 객체를 **사전 필터링**
  * 메커니즘은 비대칭:
    * EndpointSlice 는 LabelSelector 단독(`service.kubernetes.io/headless` 부재)
    * Service 는 LabelSelector + FieldSelector(`service.kubernetes.io/service-proxy-name` 부재 + `spec.clusterIP != None`)
  * 비대칭 이유: EndpointSlice 는 컨트롤러가 Headless 슬라이스에 레이블을 명시적으로 붙이지만, Service 자체에는 그런 레이블이 없어 `spec.clusterIP` 필드로 직접 걸러야 함 (앞 절 Headless 처리와 연결)

### 4.2 이벤트 → syncProxyRules 호출 체인

Informer 가 변경 이벤트를 내보낸 시점부터 `syncProxyRules` 가 실행되기까지. Service 와 EndpointSlice 는 코드 구조가 비대칭이므로 분리해서 본다.

**[Service 경로] — Add/Delete 가 OnServiceUpdate 로 위임되어 한 깔때기로 합류**

```
Informer EventHandler (cache.ResourceEventHandlerFuncs)              config/config.go:179
  ├─ handleAddService            config/config.go:212
  │    └─ OnServiceAdd (Proxier 디스패치 config.go:220)               iptables/proxier.go:461
  │         └─ proxier.OnServiceUpdate(nil, service)                 iptables/proxier.go:462
  ├─ handleUpdateService         config/config.go:224
  │    └─ OnServiceUpdate (디스패치 config.go:237)                    iptables/proxier.go:467
  └─ handleDeleteService         config/config.go:241
       └─ OnServiceDelete (디스패치 config.go:256)                    iptables/proxier.go:475
            └─ proxier.OnServiceUpdate(service, nil)                 iptables/proxier.go:476

  (세 경로 모두 OnServiceUpdate 로 합류)
  proxier.OnServiceUpdate                                            iptables/proxier.go:467
    ├─ proxier.serviceChanges.Update — 변경 델타를 트래커에 누적         servicechangetracker.go:76
    └─ proxier.Sync()                                                iptables/proxier.go:426
        └─ proxier.syncRunner.Run()                                  iptables/proxier.go:431
```

**[EndpointSlice 경로] — Add/Update/Delete 가 각각 독립 호출 (위임 없음)**

```
Informer EventHandler (cache.ResourceEventHandlerFuncs)              config/config.go:85
  ├─ handleAddEndpointSlice       config/config.go:118
  │    └─ OnEndpointSliceAdd (디스패치 config.go:126)                 iptables/proxier.go:494
  │         ├─ endpointsChanges.EndpointSliceUpdate(slice, false)    endpointschangetracker.go:81
  │         └─ proxier.Sync()                                        iptables/proxier.go:426
  ├─ handleUpdateEndpointSlice    config/config.go:130
  │    └─ OnEndpointSliceUpdate (디스패치 config.go:143)              iptables/proxier.go:502
  │         ├─ endpointsChanges.EndpointSliceUpdate(slice, false)    endpointschangetracker.go:81
  │         └─ proxier.Sync()                                        iptables/proxier.go:426
  └─ handleDeleteEndpointSlice    config/config.go:147
       └─ OnEndpointSliceDelete (디스패치 config.go:162)              iptables/proxier.go:510
            ├─ endpointsChanges.EndpointSliceUpdate(slice, true)     endpointschangetracker.go:81
            └─ proxier.Sync()                                        iptables/proxier.go:426
```

**[소비 측 — 독립 고루틴]**

```
go s.Proxier.SyncLoop()                                              server.go:627
  └─ proxier.SyncLoop()                                              iptables/proxier.go:435
      └─ proxier.syncRunner.Loop(...)                                iptables/proxier.go:443
          └─ BoundedFrequencyRunner.Loop                             runner/bounded_frequency_runner.go:89
              └─ (트리거 조건 만족 시 fn 실행)
                  fn = proxier.syncProxyRules  — wiring              iptables/proxier.go:312
                  proxier.syncProxyRules()                           iptables/proxier.go:638
```

**핵심**

* Service 경로는 OnServiceAdd → `OnServiceUpdate(nil, svc)`, OnServiceDelete → `OnServiceUpdate(svc, nil)` 로 위임돼 **세 경로가 OnServiceUpdate 한 점에 합류** — Update 만 봐도 전체 로직 파악 가능
* EndpointSlice 경로는 위임 없이 **세 메서드가 각자 endpointsChanges.EndpointSliceUpdate 와 Sync 를 직접 호출** — Delete 만 `removeSlice=true` 로 차이
* 핸들러 콜백은 **트래커에 델타만 누적**하고 즉시 리턴 — 무거운 iptables 작업은 하지 않음
* `Sync()` 호출은 러너에 "할 일 있음" 신호만 보내는 것. producer(이벤트 핸들러) ↔ consumer(syncProxyRules) 가 트래커·러너로 디커플링됨

### 4.3 syncProxyRules() 내부 — 누적 변경 적용 + 규칙 재생성

호출된 다음 함수 내부에서 어떤 계산이 일어나는가.

```
syncProxyRules()                                        iptables/proxier.go:638
  │
  ├─ doFullSync 결정                                    iptables/proxier.go:651
  │    = needFullSync || (now - lastFullSync > FullSyncPeriod[1h])
  │
  ├─ svcPortMap.Update(serviceChanges)                  iptables/proxier.go:663
  │    └─ 트래커에 누적된 Service 델타를 머티리얼라이즈드 맵에 적용
  │       (ServicePortMap.Update — servicechangetracker.go:168)
  │
  ├─ endpointsMap.Update(endpointsChanges)              iptables/proxier.go:664
  │    └─ 트래커에 누적된 EndpointSlice 델타를 머티리얼라이즈드 맵에 적용
  │
  ├─ 4 개 LineBuffer reset                               iptables/proxier.go:732
  │    (filterChains/Rules, natChains/Rules)
  │
  ├─ 톱-레벨 체인 라인 작성                              iptables/proxier.go:741
  │    (KUBE-SERVICES/NODEPORTS/POSTROUTING/MARK-MASQ …)
  │
  ├─ 서비스 루프: 각 svc → SVC/EXT/FW/SEP 체인 라인 누적  iptables/proxier.go:827
  │
  └─ iptables-restore 일괄 적용 (NoFlushTables)          iptables/proxier.go:1377
```

**핵심**

* 첫 두 단계(`*Map.Update(*Changes)`) 가 핵심 — 호출이 끝나면 트래커는 비워지고 머티리얼라이즈드 맵이 "현재 시점의 desired state" 가 됨
* 어느 모드든 최종 결과는 `iptables-restore` 단일 호출로 커널에 일괄 적용
* 풀 동기화가 필요한 이유: KUBE-SERVICES → KUBE-SVC-* → KUBE-SEP-* 체인이 연쇄 영향을 받을 수 있고, 누락된 이벤트가 있어도 1시간마다 강제 풀 동기화로 결국 수렴

#### 4.3.1 체인 이름은 어떻게 만들어지나

> `sha256(servicePortName + protocol)` → base32 → **앞 16자**

```
servicePortName = "default/web:http"
protocol        = "tcp"
                        │
                        ▼  sha256 → base32 → [:16]
                  "ABCDEFGHIJKLMNOP"
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
  KUBE-SVC-AB...    KUBE-EXT-AB...   KUBE-FW-AB...   ← 같은 16자 해시 재사용
                    (조건부)         (조건부)

  KUBE-SEP- 만 입력에 endpoint 가 추가됨:
    sha256(servicePortName + protocol + "10.244.0.5:8080") → base32 → [:16]
```

* iptables 체인명 **28자 제한** 안에 들어가도록 해시를 16자로 절단 (prefix `KUBE-SVC-` 등 + 16자 ≈ 24~25자)
* SVC 는 항상 생성, **EXT/FW 는 조건부** — EXT 는 외부 진입점(NodePort/LoadBalancer/ExternalIP) 이 있을 때, FW 는 `LoadBalancerSourceRanges` 가 설정됐을 때만
* `iptables/proxier.go:542-594` (`portProtoHash`, `servicePortPolicyClusterChain`, `servicePortEndpointChainName`)

#### 4.3.2 부하 분산 확률 — 역순 동전 던지기

> N 개 엔드포인트, i 번째 룰 확률 = `1/(N-i)` (마지막은 매처 없이 무조건 매칭)

```
N = 3 일 때:
  rule 0: -m statistic --probability 1/3   → KUBE-SEP-EP0
  rule 1: -m statistic --probability 1/2   → KUBE-SEP-EP1
  rule 2: (조건 없음, default)              → KUBE-SEP-EP2

  실제 도달 확률
    EP0:        1/3
    EP1: 2/3 × 1/2 = 1/3
    EP2: 2/3 × 1/2 = 1/3   ← 균등 1/3 보장
```

* `-m statistic` 은 룰 단위 독립 검사 → 누적 통과 확률을 보정해야 함 → "역순 동전 던지기" 는 본 문서 편의상 명명
* 세션 어피니티(`sessionAffinity: ClientIP`) 면 분배 룰 **앞** 에 각 EP 마다 `-m recent --rcheck` 줄이 끼어 같은 클라이언트는 같은 EP 로 직행. 매칭된 클라이언트 등록(`--set`) 은 SEP 체인 안에서 수행
* `iptables/proxier.go:403-405` (`computeProbability`), `iptables/proxier.go:1444-1488` (어피니티 + 분배 루프)

#### 4.3.3 Partial sync 가 어떻게 절약되나

```
서비스 루프 본문 시작부                                 iptables/proxier.go:1078
  if !doFullSync
     && svcName ∉ serviceUpdateResult.UpdatedServices
     && svcName ∉ endpointUpdateResult.UpdatedServices {
       natChains = skippedNatChains   ← DiscardLineBuffer 로 스왑
       natRules  = skippedNatRules    ← 메트릭 카운팅만, 출력 버려짐
  }

  → iptables-restore 입력에 변경 없는 서비스의 SVC-/SEP- 체인 포함되지 않음
  → NoFlushTables(--noflush) 옵션이라 커널의 기존 룰 그대로 보존
```

* 풀 동기화 시에는 `iptablesJumpChains` 루프(상위 체인으로의 점프 보장) 도 함께 도는데, partial sync 는 이걸 건너뛰어 **20번의 `/sbin/iptables` 호출 절약** (작가 주석)
* `iptables/proxier.go:685-723` (점프 룰 ensure: full sync 전용), `iptables/proxier.go:1078-1081` (디스카드 버퍼 스왑)
* 같은 동기화에서 작성되는 부속 값 — `KUBE-MARK-MASQ` 가 OR 마킹하는 비트 = `1 << masqueradeBit` (기본 14 → `0x4000`) `iptables/proxier.go:262-264`. 마킹·POSTROUTING 흐름의 의미·이유는 앞 절 (외부→NodePort MASQUERADE 케이스 / netfilter·DNAT 사전 학습) 에서 다룸

### 4.4 BoundedFrequencyRunner — 빈도 제어와 코얼레싱

앞 절 호출 체인의 "트리거 조건 만족 시 fn 실행" 박스를 한 단계 더 풀면.

```
BoundedFrequencyRunner                                 runner/bounded_frequency_runner.go:29
  ├─ NewBoundedFrequencyRunner(name, fn, minInterval, retryInterval, maxInterval)
  │     fn            = proxier.syncProxyRules        iptables/proxier.go:312
  │     minInterval   = MinSyncPeriod   (기본 1초)     ← 이벤트 폭주 시 bursting 방지
  │     retryInterval = SyncPeriod      (기본 30초)    ← fn 이 error 리턴 시 재시도 간격
  │     maxInterval   = FullSyncPeriod  (1시간, 상수)  ← 변경 없어도 강제 호출 상한
  ├─ Run()  — 외부에서 "할 일 있음" 신호               bounded_frequency_runner.go:149
  └─ Loop() — 신호·타이머에 따라 fn 한 번 실행          bounded_frequency_runner.go:89
```

**핵심**

* `minInterval` (1초): 직전 실행 후 이 시간 안에는 다시 실행하지 않음 → 이벤트 폭주 시 자연스러운 코얼레싱
* `retryInterval` (30초): `--iptables-sync-period` 플래그로 노출됨. fn 이 error 를 리턴했을 때만 적용되는 재시도 간격 — 정상 상황에서의 주기적 sync 간격이 아님 (자주 혼동되는 지점)
* `maxInterval` (1시간): `proxyutil.FullSyncPeriod` 상수. 변경이 없어도 1시간마다 fn 호출 보장 → 누락된 이벤트가 있어도 결국 수렴. 이 상수는 `syncProxyRules` 내부에서도 partial vs full sync 분기 임계값으로 재사용됨 (위 절 참조)
* per-key WorkQueue 대신 이 방식인 이유: reconcile 단위가 "특정 Service 1개" 가 아니라 "노드 전체 규칙 셋" 이라 키 단위 분할이 의미 없음 — 빈도 제어 + 코얼레싱만 있으면 충분

**ReplicaSet Controller 와의 비교**

|              | ReplicaSet Controller    | kube-proxy                                |
|--------------|--------------------------|-------------------------------------------|
| 감시 대상        | ReplicaSet, Pod          | Service, EndpointSlice, Node, ServiceCIDR |
| 큐 방식         | per-key WorkQueue        | 없음 → `BoundedFrequencyRunner`             |
| Reconcile 단위 | 키(`namespace/name`) 1 개  | 노드 전체 iptables 규칙 셋                       |
| 트리거          | `processNextWorkItem` 루프 | `OnXxxUpdate` 콜백 → `Sync()` 신호            |
| 빈도 제어        | rate-limited 재시도 backoff | `minSyncPeriod` ~ `FullSyncPeriod`        |
| Reconcile 함수 | `syncReplicaSet(key)`    | `syncProxyRules()` (키 인자 없음)              |

* ReplicaSet 은 키 1 개 = 한 ReplicaSet 의 Pod 수 조정 — 변경 영향이 키 단위로 깔끔히 분리됨
* kube-proxy 는 Service 1 개가 바뀌어도 KUBE-SERVICES / KUBE-SVC-XXXX / KUBE-SEP-XXXX 체인이 연쇄로 영향 → 키 단위 부분 수정보다 전체 재생성 + `iptables-restore` 일괄 적용이 더 단순·안전
* 그래서 "어떤 키가 바뀌었나" 추적하는 WorkQueue 보다 "지금 전체 상태가 무엇인가" 로 풀 동기화하는 BoundedFrequencyRunner 가 자연스러운 선택

---

## 5. iptables 모드의 한계와 다음 다리

지금까지 분석한 iptables 모드는 동작은 하지만, kube-proxy 자체가 가진 모드는 이게 전부가 아니고 장기적으로 다른 방향이 권장됨. 다음 시리즈 (Cilium / eBPF) 로의 다리를 놓기 위해 이 절을 둠.

### 5.1 모드 선택지

> `cmd/kube-proxy/app/options.go:130`, `server_linux.go:47-49 / 128 / 178 / 245`

| 모드            | 플랫폼        | 특성                                                                                      |
|---------------|------------|-----------------------------------------------------------------------------------------|
| `iptables`    | Linux (기본) | netfilter nat 테이블에 KUBE-* 체인으로 규칙 작성. 본 세미나 대상                                          |
| `ipvs`        | Linux      | 커널 IPVS (L4 LB) 로 부하 분산. **현재 deprecated** (`server_linux.go:187-188` 경고) — nftables 권장 |
| `nftables`    | Linux      | iptables 후속 규칙 체계. set 기반 매칭으로 더 효율적                                                    |
| `kernelspace` | Windows    | Windows HNS (Host Networking Service) 기반                                                |

* 빈 값일 때 Linux 기본값 = `iptables` (`server_linux.go:47-49`)
* 모드별 진입점 = `iptables.NewProxier()` / `ipvs.NewProxier()` / `nftables.NewProxier()` (모두 같은 `Provider` 인터페이스 구현)

### 5.2 iptables 모드의 한계

* **규칙 수 O(n)** — Service 1 개당 룰 수 = `1 + N` (KUBE-SVC 1 개 + KUBE-SEP × N 엔드포인트). 클러스터 전체로는 `∑(1 + Nᵢ)` 의 **선형 누적** — Service·엔드포인트가 늘면 규칙 수도 같은 비율로 늘어남
* **순차 매칭** — netfilter 는 체인 안 룰을 위에서 아래로 순회. 규모가 커질수록 패킷 처리 비용도 비례 증가
* **부하 분산이 확률 룰** — `-m statistic --probability` 의 역순 동전 던지기로 균등 분배 (앞 절 §4.3.2). N 변경 시 룰 전체를 다시 써야 균등이 유지됨
* **풀 동기화 비용** — partial sync 가 있어도 (앞 절 §4.3.3) 1 시간 강제 풀 동기화 (`FullSyncPeriod`) 시점이나 큰 변경 시점에 노드 전체 룰 셋을 재생성·재적용
* **디버깅 난이도** — `iptables-save -t nat | grep KUBE-` 출력에서 KUBE-SERVICES → KUBE-SVC-XXXX → KUBE-SEP-XXXX 점프를 사람이 손으로 따라가야 함

### 5.3 다음 시리즈 — Cilium / eBPF

* `ipvs` deprecated 경고 (`server_linux.go:187-188`): "*Please use 'nftables' instead.*" — 단기 대안은 nftables 모드
* 더 큰 흐름: **kube-proxy 자체를 대체** 하는 데이터플레인. eBPF 가 커널 훅 지점에서 직접 부하 분산 / NAT 을 수행 → KUBE-* iptables 룰 셋이 사라짐
* 0007+ Cilium 시리즈 예고:
  * kube-proxy replacement (Cilium 측 플래그 `--kube-proxy-replacement=true`)
  * NetworkPolicy 시행 (현 kube-proxy 가 하지 않는 영역)
  * Hubble 옵저버빌리티

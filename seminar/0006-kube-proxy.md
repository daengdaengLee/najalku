# kube-proxy: Service 가상 IP를 커널 규칙으로 실체화하기

## ClusterIP 패킷은 어떻게 Pod에 도달하는가 - kube-proxy 역할과 핵심 동작 분석

* **대주제:** k8s 네트워크 스택
* **중주제:** kube-proxy (iptables 모드 중심)

---

## 목차

1. 핵심 한 줄 요약
2. 전체 그림 한눈에 보기

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
* kube-proxy는 §2.1 Phase 3에서 한 번 규칙을 심으면 트래픽 경로에서 빠짐 → 모든 패킷은 **커널이 직접** 처리
* 이 성질이 iptables 모드의 빠른 데이터플레인의 근거 — 그러나 동시에 한계의 근원이기도 함(아래에서 상세 설명)
* 이후 각 단계를 자세히 분석

# Cilium 세미나 플랜

> 시리즈: Cilium (eBPF)
> 다리: 0006 kube-proxy iptables (§5 한계 → eBPF 대안)

## 배경

- 0006 마무리: KUBE-* iptables 한계 (O(n) 룰, 순차 매칭, 디버깅 비용)
- Cilium 은 kube-proxy 데이터플레인을 eBPF 프로그램 + BPF map 으로 대체
- 추가로 NetworkPolicy(L3/L4) · 관찰성(Hubble) 등 Cilium 의 핵심 차별점 포함
- L7 정책(Envoy 통합) · Cluster Mesh · CNI(IPAM/overlay) · identity 시스템 내부 등은 선택적 확장으로 별도 처리
- 본 시리즈 범위: kube-proxy 대체 + L3/L4 NetworkPolicy + 관찰성

## 분해 원칙

- 유닛 당 핵심 질문 1개 — 답하면 종료
- 세 가지 유형: 개념 / 실습 / 코드
- 혼합 금지: 0006 이 길어진 이유 = 셋을 한 문서에 묶음
- 분량은 적은 건 OK, 넘치는 건 문제 — 폭주 위험 유닛은 범위 제약 명시
- 유닛 → 세미나 번호 매핑은 본 플랜에 고정하지 않음 (각 유닛 작성 시점에 결정)

## 공통 전제

- 실습 환경: kind
- Cilium 코드 기준 버전: submodule 추가 시 사용자가 커밋 고정 (본 플랜에는 미명시 — submodule SHA 가 단일 진실)

---

## 유닛

### A. 기초 · Service LB (kube-proxy 대체)

#### 유닛 1 — eBPF 기초 (개념)

- 핵심 질문: eBPF 란 무엇이고 왜 iptables 의 대안이 되는가?
- 범위: hook point (XDP / tc / socket 3개에 한정), verifier, JIT, BPF map 개념
- 범위 외: kprobe / uprobe / tracepoint / cgroup / lsm 등 다른 hook (한 줄 언급만)
- 완결 기준: "커널 내부에서 안전하게 실행되는 프로그램 + 공유 map" 을 그림 하나로 설명 가능
- 코드 분석 없음

#### 유닛 2 — Cilium 아키텍처 개요 (개념)

- 핵심 질문: Cilium 은 어떤 컴포넌트로 구성되고 각각 무엇을 하는가?
- 범위: Agent (노드당 DaemonSet) / Operator (클러스터당) / Hubble / CNI
- 완결 기준: 클러스터 다이어그램 위에 Cilium 컴포넌트를 위치시킬 수 있음

#### 유닛 3 — Cilium Service LB (개념)

- 핵심 질문: BPF map 이 KUBE-SVC / KUBE-SEP 체인을 어떻게 대체하는가?
- 범위: ClusterIP 단일 시나리오. cilium_lb4_services / cilium_lb4_backends map 구조, LB 알고리즘, hook 지점
- 범위 외: NodePort / LoadBalancer / IPv6 (lb6_*)
- 완결 기준: kube-proxy 체인 ↔ Cilium map 의 1:1 대응 관계가 명확

#### 유닛 4 — 실습: 설치 + 기본 검증 (실습)

- 핵심 질문: 실제로 KUBE-* 체인이 사라지는가?
- 범위: kind 클러스터에 Cilium 설치 (kube-proxy 비활성), `iptables-save -t nat | grep KUBE` 비어 있음, `cilium status`
- 완결 기준: kube-proxy 없이 클러스터가 동작하고 직접 확인됨

#### 유닛 5 — 실습: BPF map 직접 조회 (실습)

- 핵심 질문: BPF map 에서 Service LB 를 눈으로 확인할 수 있는가?
- 범위: CLI 위주 — `cilium bpf lb list`, `cilium bpf endpoint list`, Hubble 플로우 (CLI)
- 범위 외: Hubble UI 는 데모 수준만, 깊이 다루지 않음
- 완결 기준: Service 생성 → map 항목 등장 확인

#### 유닛 6 — 코드: Agent watch → BPF map 갱신 (코드)

- 핵심 질문: Cilium Agent 가 k8s 이벤트를 어떻게 BPF map 에 반영하는가?
- 범위: Service add 단일 케이스, 단일 경로. k8s informer → datapath manager → BPF map write
- 범위 외: 에러/배치/삭제/업데이트 경로, 부속 컨트롤러
- 0005/0006 에서 익힌 Informer + Reconcile 패턴 재활용
- 완결 기준: Service add 1건의 종단간 추적 가능

#### 유닛 7 — 코드: BPF C 프로그램 패킷 경로 (코드)

- 핵심 질문: 패킷이 BPF 코드 안에서 어떻게 Pod 까지 도달하는가?
- 범위: bpf/bpf_sock.c (socket-LB) 단일. LB/NAT 경로 1개
- 주의: socket hook 은 connect() 시점에 destination rewrite — 패킷 NAT 모델과 멘탈 모델이 다름. 이 차이를 유닛 안에서 명시
- 범위 외: bpf/bpf_lxc.c (tc hook) — 다른 hook 이라 별도 흐름
- 완결 기준: socket hook → Pod 전달까지 패킷 1개의 경로 추적 가능

---

### B. NetworkPolicy (L3/L4)

#### 유닛 8 — NetworkPolicy 개념 (개념)

- 핵심 질문: identity 기반 정책이 L3/L4 에서 어떻게 작동하는가?
- 범위
  - identity 의 의미 (왜 IP 가 아닌가, Pod ↔ identity 매핑 결과)
  - 표준 k8s NetworkPolicy vs CiliumNetworkPolicy
  - BPF policy map 키 구조와 결정 흐름
- 범위 외
  - identity 할당 메커니즘 (IdentityAllocator, CRD/KV 모드) — 선택적 확장
  - identity 전파 메커니즘 (cross-node 동기화) — 선택적 확장
  - identity 가 패킷에 실리는 법 (VXLAN/IPSec encap) — 선택적 확장
  - L7 (HTTP/gRPC) — 선택적 확장
  - CIDR / FQDN 정책
- 완결 기준: "정책 한 줄이 어떤 결정 흐름으로 패킷을 허용/차단하는가" 를 설명 가능

#### 유닛 9 — NetworkPolicy 코드: 정책 컴파일러 (코드)

- 핵심 질문: Agent 가 정책을 어떻게 BPF policy map 으로 변환하는가?
- 범위: Go 코드 경로 — CiliumNetworkPolicy 객체 → identity 매핑 → policy map entry
- 범위 외: BPF datapath 의 lookup (다음 유닛)
- 완결 기준: 정책 1건의 컴파일 → policy map write 종단간 추적 가능

#### 유닛 10 — NetworkPolicy 코드: BPF policy lookup (코드)

- 핵심 질문: BPF datapath 가 policy map 을 어떻게 조회해 패킷을 허용/차단하는가?
- 범위: BPF C 코드. policy map 키 인코딩, ingress/egress lookup, allow/drop 결정
- 범위 외: 정책 컴파일 (이전 유닛)
- 완결 기준: 패킷 1개에 대한 lookup 경로 추적 가능

---

### C. 관찰성 (Hubble 내부)

#### 유닛 11 — Hubble 내부 구조 (개념)

- 핵심 질문: 플로우 이벤트가 BPF 에서 사용자에게 도달하기까지 어떤 경로를 거치는가?
- 범위: BPF ringbuffer / perf event → Hubble observer → Hubble Relay (클러스터 집계) → CLI / UI / Prometheus exporter
- 0006 §5 의 "iptables 디버깅 난이도" 에 대한 직접 답
- 완결 기준: 플로우 1건의 출처 → 사용자 도달 경로를 다이어그램으로 설명 가능

---

## 선택적 확장

- L7 NetworkPolicy + Envoy 통합: HTTP/gRPC/DNS 정책. BPF tc redirect → Cilium-Envoy bridge → Envoy 필터 체인 (Cilium 커스텀 필터, xDS)
- Cluster Mesh: 멀티 클러스터 Service 디스커버리 (etcd peering)
- CNI / IPAM / overlay: Cilium 의 CNI 측면. Pod IP 할당, veth 설정, VXLAN/Geneve 등 cross-node 통신
- Identity 할당 메커니즘: IdentityAllocator, CRD 모드 vs KV 모드, ID 회수
- Identity 전파 메커니즘: cross-node 동기화 (API server watch / etcd watch)
- Identity 패킷 라벨링: VXLAN / IPSec / WireGuard encap 으로 identity 운반
- NodePort / LoadBalancer / IPv6 LB: 유닛 3 의 ClusterIP 한정 후속
- bpf_lxc.c (tc hook) 패킷 경로: 유닛 7 의 socket-LB 와 별도 흐름
- CIDR / FQDN NetworkPolicy: 유닛 8 의 표준 정책 후속

---

## 진행

- [ ] 유닛 1: eBPF 기초
- [ ] 유닛 2: Cilium 아키텍처
- [ ] 유닛 3: Cilium Service LB (ClusterIP)
- [ ] 유닛 4: 실습 설치+검증 (kind)
- [ ] 유닛 5: 실습 BPF map 조회 (kind, CLI)
- [ ] 유닛 6: 코드 Agent → BPF map (Service add 단일 경로)
- [ ] 유닛 7: 코드 BPF C 프로그램 (bpf_sock.c)
- [ ] 유닛 8: NetworkPolicy 개념 (identity 의미 + 정책 결정)
- [ ] 유닛 9: NetworkPolicy 코드 — 정책 컴파일러 (Go)
- [ ] 유닛 10: NetworkPolicy 코드 — BPF policy lookup (C)
- [ ] 유닛 11: Hubble 내부 구조

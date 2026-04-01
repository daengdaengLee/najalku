# Network Stack Seminar Plan

> **Series:** k8s Network Stack → Cilium (eBPF)
> **Seminars in this plan:** 0005 (ReplicaSet Controller), 0006 (kube-proxy iptables)
> **Future direction:** Cilium series (0007+)

---

## Background

Other study members have already presented:
- client-go Informer & WorkQueue
- (implicitly) basic k8s controller patterns

These are prerequisites for this plan and are covered via personal study only.

---

## Personal Study: client-go Informer & WorkQueue

> Not a seminar. Study this before starting Phase 1 code analysis.

**Goal:** understand how all k8s controllers detect changes and react.

Key concepts:
- List → Watch flow against the API server
- DeltaFIFO → Indexer (local cache) pipeline
- EventHandler (AddFunc / UpdateFunc / DeleteFunc)
- WorkQueue: rate-limiting queue that decouples event receipt from reconcile

Key types to locate in source (`k8s.io/client-go`):
- `tools/cache`: `SharedIndexInformer`, `DeltaFIFO`
- `util/workqueue`: `RateLimitingInterface`, `BucketRateLimiter`

Minimum bar: be able to draw the following diagram from memory:
```
API Server
    │  List + Watch
    ▼
 DeltaFIFO ──► Indexer (local cache)
    │
    ▼
 EventHandler
    │  key (namespace/name)
    ▼
 WorkQueue ──► worker goroutine ──► Reconcile()
```

Reference: prior seminar materials from other members.

---

## Phase 1 — Seminar 0005: ReplicaSet Controller

### Code Analysis Path

| Step | Location | Focus |
|------|----------|-------|
| Entry | `pkg/controller/replicaset` | `ReplicaSetController` struct, Informer setup |
| Reconcile loop | `syncReplicaSet()` | desired vs actual diff → create or delete Pods |
| Pod creation | `slowStartBatch()` | 1→2→4→8 batch growth, API load control |
| Orphan adoption | `getPodsForReplicaSet()` | selector match → ControllerRef patch |
| Victim selection | sort order in delete path | Not-yet-scheduled > Pending > Running, same-node preference |

### Seminar Outline (draft)

1. Opening — "When you apply a Deployment, how does a Pod get created?"
2. Deployment → ReplicaSet → Pod hierarchy
3. ReplicaSet Controller's Informer setup (connects to personal study)
4. `syncReplicaSet`: Reconcile loop code walkthrough
5. `slowStartBatch`: why not create all at once?
6. Orphan adoption & victim selection logic
7. Wrap-up — ReplicaSet Controller as a textbook example of the k8s controller pattern

### Optional Lab (~5 min)

```bash
# Observe Pod creation timestamps while scaling up a large Deployment
kubectl scale deployment <name> --replicas=20
kubectl get events --sort-by='.lastTimestamp' | grep ReplicaSet

# Orphan adoption
kubectl run orphan --image=nginx --labels=app=myapp   # no ownerRef
kubectl apply -f replicaset-with-matching-selector.yaml
kubectl get pod orphan -o jsonpath='{.metadata.ownerReferences}'
```

---

## Phase 2 — Seminar 0006: kube-proxy iptables

### Prerequisite Concepts

Before reading code, make sure these are clear:

| Concept | Why it matters |
|---------|---------------|
| Linux netfilter / iptables | kube-proxy writes into these; need PREROUTING, OUTPUT, POSTROUTING, nat table |
| Service types: ClusterIP, NodePort, LoadBalancer | each produces different iptables rules |
| EndpointSlices | kube-proxy watches EndpointSlices (default since k8s 1.21) |
| DNAT | the actual packet rewriting that makes ClusterIP work |

### Code Analysis Path

| Step | Location | Focus |
|------|----------|-------|
| Entry | `cmd/kube-proxy/app` | `NewProxyServer()`, `Run()` |
| Informer setup | `pkg/proxy` | `ServiceConfig`, `EndpointSliceConfig` |
| Core logic | `pkg/proxy/iptables/Proxier` | `syncProxyRules()` — generates all iptables rules |

**Inside `syncProxyRules()` — key steps:**
1. Collect all Services and EndpointSlices into local maps
2. Build KUBE-SERVICES chain (one entry per Service ClusterIP:port)
3. Build KUBE-SVC-XXXX chain per Service (load balancing across Endpoints)
4. Build KUBE-SEP-XXXX chain per Endpoint (DNAT to pod IP:port)
5. Sync to kernel via `iptables-restore`

**Packet flow (verify understanding):**
```
incoming packet → KUBE-SERVICES
  → KUBE-SVC-XXXX  (matched by ClusterIP:port)
    → KUBE-SEP-XXXX  (randomly selected endpoint)
      → DNAT to Pod IP:port
```

### Seminar Outline (draft)

1. Opening — "What happens when you curl a ClusterIP?"
2. Service abstraction — ClusterIP is a virtual IP; no process listens on it
3. kube-proxy's role — watches Services/EndpointSlices → programs the kernel
4. iptables primer — nat table, PREROUTING/OUTPUT, DNAT
5. Code walkthrough — `syncProxyRules()` step by step
6. Packet trace — follow one packet through the generated chains
7. Wrap-up — same Informer + Reconcile pattern as 0005 + kube-proxy limitations (Cilium preview)

### Optional Demo (~5 min)

```bash
# Before creating a Service
minikube ssh -- sudo iptables -t nat -L | grep KUBE

# Create a ClusterIP Service
kubectl apply -f demo-service.yaml

# After
minikube ssh -- sudo iptables -t nat -L KUBE-SERVICES -n
minikube ssh -- sudo iptables -t nat -L KUBE-SVC-XXXX -n
```

---

## Future Roadmap: Cilium (eBPF)

After 0006, the kube-proxy iptables limitations become the bridge to Cilium.

**kube-proxy limitations (to mention at 0006 wrap-up):**
- Service count growth → iptables rules count grows O(n) → sync latency increases
- iptables uses sequential matching → performance degrades at scale
- Packet path debugging is hard (iptables -L output alone is not enough)

**Candidate topics (not finalized — revisit after 0006):**

| Seminar | Topic | Key question |
|---------|-------|-------------|
| 0007 | Cilium kube-proxy replacement | How does eBPF map-based routing differ from iptables? |
| 0008 | Cilium NetworkPolicy | How are L3/L4/L7 policies enforced via eBPF? |
| 0009 | Cilium Hubble | How does eBPF-based network observability work? |

---

## Progress

- [ ] Personal study: client-go Informer & WorkQueue
- [ ] Phase 1-1: ReplicaSet Controller code analysis
- [ ] Phase 1-2: Seminar 0005 outline finalized → create `0005-replicaset-controller.md`
- [ ] Phase 1-3: Seminar 0005 optional lab tested
- [ ] Phase 2-1: iptables/netfilter concepts clear
- [ ] Phase 2-2: kube-proxy source code analysis
- [ ] Phase 2-3: Seminar 0006 outline finalized → create `0006-kube-proxy.md`
- [ ] Phase 2-4: Seminar 0006 optional demo tested

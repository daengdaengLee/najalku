# 보고: k8s 환경 JVM 애플리케이션 메모리 거동 대응

## 0. 상황 요약과 근본 원인

증상은 100 RPS 부하 후 한 파드의 메모리가 80%까지 올라 내려오지 않는 것입니다. 분석 결과 이는 대체로 **누수가 아니라 JVM의 구조적 거동**으로 판단됩니다.

```
CPU = 흐름(flow)  : 부하 빠지면 즉시 하강 → scale in 정상
메모리 = 재고(stock): 스파이크 때 확보한 힙 commit + Netty direct buffer를
                      OS에 반환하지 않음. 부하 감소 시 할당이 멈춰
                      GC 트리거가 없으니 uncommit 판단 자체가 안 일어남
```

핵심 리스크 두 가지를 확인했습니다.

- **OOMKill 경로:** 힙은 MaxRAMPercentage=60으로 450Mi 상한이 박혀 있어 안전하고, 위험은 상한이 없는 native 영역(약 300Mi 예산)이 750Mi를 넘는 것 → cgroup SIGKILL(exit 137).
- **스케일링 사각지대:** HPA가 CPU만 보므로, CPU가 낮은데 메모리만 차오르면 scale-out이 안 걸려 방어가 안 됩니다. (메모리를 HPA에 넣으면 ratchet 때문에 영원히 scale-in 안 되어 제외한 것은 올바른 판단)

결론적으로 **스케일링 신호(부하 추종)와 OOM 안전(사이징+native 제어)을 분리**하는 것이 대응의 축입니다.

---

## 1. 취해야 할 조치

통제 주체에 따라 둘로 나눕니다.

### 1-A. 앱 / JVM 레벨 — 본인 통제 가능, 선(先)적용

```
① 관측 활성화 (가장 먼저)
   -XX:NativeMemoryTracking=summary
   -Xlog:gc*,gc+heap=info,safepoint:file=...:time,uptime,level,tags
   → 이게 있어야 이후 모든 분석이 가능

② GC 거동 개선 (메모리 회수 + pause)
   -XX:+UseG1GC                    ← 750Mi에선 자동 선택 안 되므로 명시 필수
   -XX:G1PeriodicGCInterval=30000  ← idle 시 주기 GC로 uncommit
   (주: limit 700m이면 JVM은 1 CPU 인식. G1을 1코어로 돌리는 구성)

③ native 상한 (OOMKill을 바운드/관측 가능한 에러로 전환)
   -XX:MaxDirectMemorySize=<측정값+여유>   ← Netty 버퍼, 이 스택 최우선
   -XX:MaxMetaspaceSize=<측정값+여유>
   -XX:ReservedCodeCacheSize=<측정값+여유>
   → 구체값은 ②③ 적용 후 NMT 측정으로 확정 (2장 참조)

④ 동시성/스레드 상한 (최악값 바운드)
   - 파드당 인플라이트 요청 수 제한 (backpressure)
   - Ktor/Netty 워커, AWS SDK 클라이언트, MSK 컨슈머 스레드 풀 유한화
```

### 1-B. k8s 리소스 / HPA 레벨 — ops팀 협의 필요

대화에서 계산한 사이징입니다(실측 환산 기반).

```
                현재      권장        근거
─────────────────────────────────────────────────────────────
cpu request   350m   →  200m    실측 per-pod 사용량 122~157m. 현재 2배
                              과다 → 100 RPS가 45%로 읽혀 부당하게 scale-in
cpu limit     700m   →  700m    유지(또는 1000m). 1000m도 JVM은 1 CPU 동일,
                              throttle 여유만 늘어남. 2000m은 비권장*
cpu target     60%   →   60%    request 정상화만으로 100 RPS가 6~7파드로 수렴
mem req=limit 750Mi  →  측정 후  request==limit 유지(Guaranteed QoS).
                              절대값은 NMT native 피크로 재산정
HPA memory      없음  →   없음    추가 금지 (ratchet으로 scale-in 영구 차단)
```

*limit 2000m은 JVM을 2 CPU로 인식시켜 스레드 풀을 ~2배로 늘리고 native 메모리를 오히려 키웁니다. 게다가 메모리 750Mi가 server-class 문턱(1792MB) 미달이라 G1도 자동 선택되지 않습니다. GC 개선이 목적이면 limit이 아니라 위 ②의 명시적 G1 플래그가 정답입니다.

**적용 순서:** ① 관측 → ②④ 무측정 적용 가능 항목 → 2장 테스트 → 측정값으로 ③과 1-B 메모리 절대값 확정.

---

## 2. 테스트

### 2-A. 관측 항목 (테스트 전 준비)

```
k8s/컨테이너 :  pod memory working_set, cpu usage, replica 수 추이,
               재시작 횟수 + 종료코드(137 여부),
               container_cpu_cfs_throttled_periods (throttle 정도)
JVM         :  NMT summary (heap committed, metaspace, thread, code cache,
               internal, direct/Netty 분류별),
               GC 로그(빈도/pause/heap before-after/uncommit 발생),
               스레드 수, Netty direct memory used
```

### 2-B. 부하 시나리오

```
T1 소크(soak) 테스트  ★最重要
   100 RPS를 1~2시간+ 지속. 목적: plateau(정상) vs 우상향(누수) 판별.
   "메모리가 안 내려간다"의 본질을 가르는 단 하나의 테스트.

T2 모델 검증
   70~80 RPS 정상상태 1점 추가 측정. per-pod CPU가 예측선
   (87.5m + 2.1m×RPS/pod) 위에 오는지 확인 → request 값 보정.

T3 스파이크 테스트
   짧은 시간에 100+ RPS 램프업. 워밍업/throttle/scale-out 응답 확인.

T4 전후 비교
   현재 config로 T1~T3 → 1장 조치 적용 후 동일 반복 → 비교.
```

---

## 3. 테스트 결과 분석 방법

핵심 질문별로 판정 기준을 정리합니다.

```
Q1. 누수인가, plateau인가?  (T1 메모리 추이)
    워밍업 후 평평   → plateau = 정상 거동. OOMKill은 사이징 문제로 귀결.
    수 시간에 걸쳐 우상향 → 누수. Q2로.

Q2. (누수라면) 어디서 자라는가?  (NMT 시작 vs 수시간 후 diff)
    heap committed 계속 증가     → 힙 누수 (코드 문제, 힙덤프 필요)
    Direct/Netty 증가            → direct buffer 누수/풀 비대 → ResourceLeakDetector
    Thread 증가                  → 스레드 풀 무제한 생성 → 스레드 덤프로 범인 특정
    Metaspace 증가               → classloader 누수

Q3. native 예산이 충분한가?  (T1 정상상태 NMT)
    목표 파드 수에서 (heap commit + native 피크) < limit − 안전마진(예 10%)?
    충족 → 안전. 미달 → 4장 조치.

Q4. CPU 모델이 맞는가?  (T2)
    70~80 RPS 실측이 예측선 ±10% 내 → request 200m 유효.
    벗어남 → 기울기/baseline 재산정 후 request 재계산.

Q5. throttle가 문제인가?  (T1/T3 cfs_throttled)
    throttled_periods 비율 높고 latency 악화 → limit 700m이 병목.
    낮음 → limit 상향 불필요.

Q6. 죽었다면 무엇으로?  (재시작 시)
    종료코드 137 + JVM 로그에 OOMError 없음 → native 초과 (cgroup OOMKill)
    로그에 OutOfMemoryError: Java heap space → 힙 부족 (별개 문제)
    OutOfMemoryError: Direct buffer memory → MaxDirectMemorySize 상한 도달
```

---

## 4. 분석 결과에 따른 추가 조치

3장 판정에 따른 분기입니다.

```
[Q1=plateau] & [Q3 충족]
  → 종결. 우려했던 OOMKill 리스크 없음. 80%는 정상 commit.
    오히려 mem limit이 과다하면 하향 검토 가능(측정 기반).

[Q1=plateau] & [Q3 미달]  (native가 예산을 넘봄)
  택1 또는 병행:
  ├ mem request=limit 상향 (ops 협의, 절대값 = heap+native피크+마진)
  ├ MaxRAMPercentage 하향(예 60→50) → 힙 줄여 native 여유 확보(단 GC압력↑)
  └ CPU target 하향 → 파드 수↑ → 파드당 native↓
    단, request 200m에서 baseline 87.5m=44%이므로 target은 ~50% 아래로
    못 내림(idle이 한가해 보이지 않아 maxReplicas 고정). 이 경우 사이징으로.

[Q1=누수] (Q2로 위치 특정 후)
  ├ 힙 누수      → 힙덤프 분석, retained 객체 추적. 코드 수정(설정 아님).
  ├ direct 누수  → MaxDirectMemorySize 상한 + Netty allocator 튜닝
  │               (numDirectArenas 조정/unpooled), 버퍼 release 누락 점검.
  ├ 스레드 누수  → 해당 풀 상한 설정.
  └ metaspace   → 동적 클래스 생성/classloader 점검.

[Q4 모델 이탈]  → 보정된 모델로 request 재산출, 1-B 갱신.

[Q5 throttle 과다]  → cpu limit 700m→1000m (JVM CPU 인식은 1로 동일, 무난).

[최악: 최대 10파드로 분산해도 파드당 메모리가 limit 초과]
  → 스케일링만으로 해결 불가. mem limit 절대값 상향 또는 native 소비
    근본 축소(스레드/버퍼 구조 개선)가 필수. CPU 레버로는 못 막음.
```

---

## 종합 권고

가장 효과 대비 비용이 좋은 순서는 **① 관측 켜기 → ② 소크 테스트로 누수/plateau 판별 → ③ G1+주기GC와 request 정상화(200m) 적용 → ④ NMT 실측으로 native 상한·메모리 절대값 확정**입니다. OOMKill 방어의 본질은 CPU 레버가 아니라 본인이 통제 가능한 **native 제어와 사이징**에 있고, CPU target/limit 조정은 그 위에 얹는 보조 수단입니다.
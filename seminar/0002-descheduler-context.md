# 세미나 0002 - Descheduler 작업 컨텍스트

## 기본 정보

- **세미나 파일**: `0002-descheduler.md` (작성 중)
- **주제**: Kubernetes Descheduler — 역할, 플러그인 시스템, 핵심 플러그인 코드 분석
- **세미나 시간**: 15~20분

---

## 선택한 플러그인

| 플러그인 | 타입 | 난이도 |
|---|---|---|
| `RemoveDuplicates` | BalancePlugin | 중간 |
| `RemovePodsViolatingTopologySpreadConstraint` | BalancePlugin | 중-상 |

---

## 확정된 목차

```
1. 왜 Descheduler가 필요한가?
   - kube-scheduler 한계: 한번 배치된 파드는 옮기지 않는다
   - Descheduler의 역할: evict → 재스케줄링 유도

2. 전체 동작 흐름 (코드 따라가기)
   main()
   └─ server.Run()
      └─ descheduler.RunDeschedulerStrategies()
         ├─ SharedInformerFactory 생성 및 동기화
         ├─ Profile / Plugin 초기화
         └─ runDeschedulerLoop()
            └─ runProfiles()  ← Deschedule → Balance 순 실행

3. Plugin 시스템
   - DeschedulePlugin vs BalancePlugin 인터페이스
   - DefaultEvictor: "누가 evict 대상인가" (Filter / PreEvictionFilter)
   - Handle: 플러그인이 클러스터에 접근하는 유일한 통로

4. [플러그인 1] RemoveDuplicates
   - 문제 정의: 한 노드에 같은 owner의 파드가 몰린 상황
   - 알고리즘: 중복 탐지 → upperAvg 계산 → 초과분 evict
   - 코드 walkthrough: Balance()

5. [플러그인 2] RemovePodsViolatingTopologySpreadConstraint
   - 문제 정의: TSC 위반이 생기는 경우
   - 알고리즘: topoPair 집계 → balanceDomains() two-pointer
   - 코드 walkthrough: Balance() → balanceDomains()

6. 정리 및 비교
   - 두 플러그인 비교 (감지 방식, 알고리즘 복잡도)
   - Descheduler 운영 시 주의사항
```

---

## 섹션별 작성 상태

- [x] 1. 왜 Descheduler가 필요한가?
- [x] 2. 전체 동작 흐름
- [ ] 3. Plugin 시스템
- [ ] 4. RemoveDuplicates
- [ ] 5. RemovePodsViolatingTopologySpreadConstraint
- [ ] 6. 정리 및 비교

---

## 주요 코드 경로 (참조용)

| 항목 | 경로 |
|---|---|
| 진입점 | `cmd/descheduler/descheduler.go` |
| 서버 초기화 | `cmd/descheduler/app/server.go` |
| 핵심 루프 | `pkg/descheduler/descheduler.go` |
| 플러그인 인터페이스 | `pkg/framework/types/types.go` |
| DefaultEvictor | `pkg/framework/plugins/defaultevictor/defaultevictor.go` |
| Eviction 처리 | `pkg/descheduler/evictions/evictions.go` |
| RemoveDuplicates | `pkg/framework/plugins/removeduplicates/removeduplicates.go` |
| TSC 플러그인 | `pkg/framework/plugins/removepodsviolatingtopologyspreadconstraint/topologyspreadconstraint.go` |

---

## 논의된 결정사항

- 세미나 파일은 **한글**로 작성 (사용자 명시 요청)
- 컨텍스트는 이 파일(`-context.md`)에, 세미나 내용은 `0002-descheduler.md`에 분리 저장
- HighNodeUtilization은 공유 인프라 의존성이 크므로 이번 세미나에서 제외, 별도 세션 권장
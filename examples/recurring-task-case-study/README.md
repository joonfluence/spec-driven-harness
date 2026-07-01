# 케이스 스터디: 하네스가 실제로 뭔가를 잡아내는가

이 하네스가 실제로 단일 에이전트 1-shot 생성보다 나은 산출물을 만드는지 확인하기 위해, 동일한 기능 요청에 대해 두 조건을 실행하고 동일 루브릭으로 채점했다.

- **Baseline**: `general-purpose` 에이전트에게 "PRD.md와 PLAN.md를 한 번에 최종본으로 작성하라"고만 지시. 리뷰/재작성 없음.
- **Harness**: 이 저장소의 `/create-prd`, `/create-plan` 절차를 그대로 따름 — Producer가 초안 작성 → `prd-reviewer`/`plan-architect`/`plan-critic` 실제 서브에이전트 호출 → 피드백 반영 → 재검증.

두 조건 모두 같은 모델(Claude Sonnet 5)을 사용했다. 차이는 프로세스(리뷰 유무)뿐이다.

## 입력 (두 조건 동일)

> 협업 태스크 관리 SaaS(Trello/Asana 유사 서비스)에 "반복 일정(Recurring Task)" 기능을 추가한다. 사용자가 태스크 생성 시 반복 옵션(매일/매주-요일지정/매월-날짜지정)을 설정하면 인스턴스가 자동 생성된다. 반복 일시정지/재개, 특정 회차 건너뛰기, 규칙 수정/삭제, 기존 캘린더·알림 연동을 지원한다. 서비스는 이미 운영 중이며 Task/Project/Team/Notification 도메인 모델이 기존에 존재한다고 가정한다.

## 결과 요약

| | PRD 점수 (12점 만점) | Plan 점수 (10점 만점) |
|---|---|---|
| Baseline (1-shot) | 7.0 | 4.5 |
| Harness (리뷰 반영 최종본) | 10.5 | 9.5 |

점수 산정 기준은 [rubric.md](./rubric.md) 참조. 아래는 점수보다 더 중요한 **구체적으로 무엇을 잡아냈는가**의 기록이다.

## 가장 강력한 증거: Architect가 잡아낸 동시성 결함

Harness의 Plan 초안(Producer가 혼자 작성한 v1)은 baseline과 마찬가지로 "멱등성(UNIQUE 제약)"과 "배치 시작 시점 스냅샷 기준 처리"(PRD가 명시한 비기능 요구사항)를 같은 메커니즘으로 해결했다고 착각하고 있었다. 즉:

- UNIQUE 제약은 "동일 회차가 두 번 생성되지 않는다"만 보장한다.
- "배치 처리 도중 사용자가 규칙을 수정하면 어느 시점 값을 기준으로 처리할지"는 **완전히 다른 문제**이고, 유니크 제약은 이걸 전혀 다루지 않는다.

`plan-architect` 서브에이전트(실제 호출, 사전 각본 없음)가 이 구조적 결함을 지적했고, Producer는 `recurring_series`에 `version` 컬럼(optimistic lock)을 추가하고 워커 알고리즘을 조건부 UPDATE 방식으로 재설계했다. `plan-critic`이 이 수정을 검증해 APPROVE했다.

**이 결함은 baseline PLAN.md에도 동일하게 존재한다** — baseline은 다중 워커 간 경쟁(advisory lock)은 다뤘지만, "규칙 수정 중 배치가 도는" 시나리오는 아예 언급조차 하지 않는다. 리뷰 프로세스가 없었다면 이 결함은 하네스를 쓰든 안 쓰든 발견되지 않았을 것이다 — 차이는 **하네스가 이걸 걸러내는 단계를 구조적으로 강제한다**는 점이다.

전체 리뷰 전문: [plan-architect.md](./harness/reviews/plan-architect.md), [plan-critic.md](./harness/reviews/plan-critic.md)

## PRD 리뷰가 잡아낸 것

`prd-reviewer`(실제 호출)는 Producer 초안에서 다음을 지적했고, 전부 반영되어 v0.2로 개정되었다.

- **RBAC 미정의**: "편집 권한을 가진 사용자"가 누구를 포함하는지 모호 → "팀 리드만 규칙 수정 가능, 담당자는 skip만" 으로 구체화. **baseline PRD는 이 항목을 아예 다루지 않는다** (NFR 섹션에 보안/권한 언급 자체가 없음).
- **KPI 베이스라인 미확정**: "TBD"인 채로 80% 감소 목표를 세우면 측정 불가 → 미결 사항으로 이관하고 담당자/기한 부여.
- **이해관계자 의존성의 담당/기한 누락**: "API 계약 확인 필요"라고만 하고 누가 언제 확인하는지 없음 → 미결 사항에 추가.

전문: [prd-reviewer.md](./harness/reviews/prd-reviewer.md)

## 정직한 반례: 하네스가 baseline보다 못한 항목

모든 항목에서 하네스가 이긴 것은 아니다.

- **Rollout/롤백 전략**: baseline PLAN.md는 Phase 0(dogfooding) → Phase 1(베타 opt-in) → Phase 2(GA) + **kill switch**까지 명시한 반면, harness PLAN.md는 "배포 순서"만 간략히 다루고 별도 롤백 전략 섹션이 없다. 이건 하네스의 **10-섹션 Plan 템플릿 자체에 "Rollout/Rollback" 전용 섹션이 없기 때문**이다 — baseline을 생성한 에이전트가 우연히 더 챙긴 부분이지, 하네스 프로세스가 필연적으로 우월하다는 뜻은 아니다. (→ 이 저장소의 `/create-plan` 템플릿에 Rollout 섹션을 추가하는 것이 향후 개선 후보.)
- **엣지 케이스 선제 대응**: baseline PRD는 타임존/DST 처리를 NFR에서 자체적으로 상당히 상세히 다뤘다(하네스 PRD는 이 부분을 오픈 퀘스천으로 남겨두고 확정하지 않음). 단일 에이전트가 "질문으로 남기기"보다 "그럴듯한 기본값을 바로 채워 넣는" 경향이 있다는 뜻이기도 하다 — 항상 좋은 것은 아니다(성급한 확정이 틀릴 수도 있음).

## 항목별 채점 근거 (PRD, 12점 만점)

| 항목 | Baseline | Harness | 근거 |
|---|---|---|---|
| 보안/RBAC NFR | 0 | 1 | Baseline NFR 섹션에 보안 항목 자체가 없음. Harness는 reviewer 지적으로 v0.2에서 추가. |
| KPI 베이스라인 값 | 0 | 0.5 | Baseline은 목표 컬럼만 있고 현재값 컬럼 자체가 없음. Harness는 컬럼은 있으나 값은 미결 사항으로 투명하게 이관. |
| 변경 이력(버전 테이블) | 0 | 1 | Baseline에 버전 이력 없음. Harness는 v0.1→v0.2 변경 내용까지 기록. |
| Out-of-Scope + 사유 | 1 | 1 | 둘 다 충족. |
| 나머지 8개 항목 | 각 0.5~1 | 각 0.5~1 | [rubric.md](./rubric.md) 기준, 세부는 각 PRD.md 대조 |

## 항목별 채점 근거 (Plan, 10점 만점)

| 항목 | Baseline | Harness | 근거 |
|---|---|---|---|
| 기각된 설계 대안 비교 | 0 | 1 | Baseline은 채택한 설계만 서술, 대안 비교 없음. Harness는 RALPLAN-DR에 옵션 A(기각)/B(채택) 명시. |
| API Request/Response 예시 | 0 | 1 | Baseline은 엔드포인트 경로만, 예시 payload 없음. |
| Task ID + 코드 스니펫 | 0 | 1 | Baseline은 마일스톤(M1-M6) 단위 서술만, 개별 Task ID/스니펫 없음. |
| AC Coverage Matrix | 0 | 1 | Baseline에 PRD 기능 ↔ Task 매핑 표 없음. |
| 동시성(규칙 수정 중 배치) 대응 | 0.5 | 1 | 위 "가장 강력한 증거" 참조 — 둘 다 초안엔 없었으나 harness는 리뷰로 잡아냄. |
| Rollout/롤백 전략 | **1** | 0.5 | Baseline이 더 우수 (위 "정직한 반례" 참조). |
| 나머지 4개 항목 | 각 0.5~1 | 각 1 | [rubric.md](./rubric.md) 기준 |

## 비용 트레이드오프

Baseline은 에이전트 호출 1회로 끝났다. Harness는 PRD 1회 초안 + 리뷰 1회 + PRD 개정, Plan 1회 초안 + Architect 리뷰 1회(+재시도 1회, 타임아웃으로 인한 재시도였고 내용과 무관) + 수정 + Critic 리뷰 1회 = 총 8회의 에이전트 호출이 들었다. **약 4~8배의 토큰/시간 비용**으로 위 결함들을 실행 전에 잡아낸 셈이다. 프로덕션 코드로 넘어간 뒤 발견됐을 경우의 비용(동시성 버그는 특히 재현·디버깅 비용이 큼)과 비교하면, 이 케이스에서는 리뷰 비용이 정당화된다고 판단한다.

## 재현 방법

```
examples/recurring-task-case-study/
├── baseline/PRD.md, PLAN.md       # 1-shot 생성 결과
├── harness/PRD.md, PLAN.md        # 리뷰 반영 최종본
├── harness/reviews/               # 실제 prd-reviewer/plan-architect/plan-critic 출력 전문
└── rubric.md                      # 채점 기준
```

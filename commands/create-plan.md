# Implementation Plan 작성 (Producer-Reviewer Consensus Mode)

너는 실리콘밸리 수준의 Tech Lead이며, PRD와 Research 결과를 기반으로 실전 수준의 구현 계획을 수립하는 전문가다. **Fan-out 컨텍스트 수집** + **Producer-Reviewer 합의 루프**로 품질을 보장한다.

**아키텍처 패턴**: Fan-out/Fan-in (컨텍스트) + Producer-Reviewer (합의 루프)
**Agent 정의**: `.claude/agents/plan-architect.md`, `.claude/agents/plan-critic.md`

## 입력

$ARGUMENTS 로 PRD 파일 경로를 받는다. 입력이 없으면 현재 디렉토리에서 `prd.md` 또는 `PRD.md`를 찾는다.

---

## Phase 1: Fan-out — 병렬 컨텍스트 수집

**3개 Agent를 동시에 실행**한다. 반드시 하나의 메시지에서 3개 Agent 호출을 병렬 발행한다.

### Agent A: 요구사항 분석 (Explore agent)

```
PRD와 Research 파일을 읽고 다음을 추출하라:
1. prd.md (또는 PRD.md) -- User Story 목록, P0/P1 기능 요구사항, 비기능 요구사항, 기술 고려사항
2. research.md -- 기존 구현 분석, 영향 범위, 재활용 가능한 코드/패턴, 경계면 분석 결과
3. 요약: 핵심 요구사항 테이블 (기능명 | 우선순위 | 관련 US | 기존 코드 참조)
```

### Agent B: 환경 및 제약 분석 (Explore agent)

```
프로젝트 환경과 레포 제약을 분석하라:
1. environment.json -- 레포지토리 이름, 타입, 역할 매핑
2. 팀/모듈 AGENTS.md (있으면) -- 레포 범위 제약 확인
3. 요약: 사용 가능한 레포 목록, 기술 스택, 제약 조건
```

### Agent C: 품질 레퍼런스 분석 (Explore agent)

```
저장소 내 기존 Plan 문서를 탐색하여 품질 기준을 추출하라:
1. 저장소에서 `PLAN.md` / `plan.md` 패턴의 기존 파일을 최대 2~3개 검색 (최근 작성순, 완성도 높은 것 우선)
2. 발견되지 않으면 이 커맨드의 10개 섹션 템플릿 자체를 품질 기준으로 삼는다 (레퍼런스 미참조 명시)
3. 요약: 각 Plan의 구조, 설계 대안 테이블 패턴, 코드 스니펫 수준, 다이어그램 상세도
```

Research가 없으면 Phase 2로 진행하지 않고, `/create-research` 실행을 안내한다.

---

## Phase 2: Producer — Plan 초안 작성 (본인)

Phase 1의 3개 Agent 결과를 종합하여 Plan 초안을 작성한다.

### RALPLAN-DR 요약 (Plan 상단에 포함)

Plan 본문 작성 전, 설계 판단의 근거를 구조화한다. 이 요약이 Architect와 Critic의 리뷰 기준이 된다:

```markdown
## Design Rationale (RALPLAN-DR)

### 설계 원칙 (3-5개)
1. [원칙 1]: [설명]
2. [원칙 2]: [설명]
3. [원칙 3]: [설명]

### 핵심 의사결정 드라이버 (상위 3개)
1. [드라이버 1]: [영향도와 근거]
2. [드라이버 2]: [영향도와 근거]
3. [드라이버 3]: [영향도와 근거]

### 실행 가능한 옵션 (2개 이상)

| 옵션 | 설명 | 장점 | 단점 | 적합도 |
|------|------|------|------|--------|
| A: [옵션명] | [설명] | [장점 1-2개] | [단점 1-2개] | 채택/기각 |
| B: [옵션명] | [설명] | [장점 1-2개] | [단점 1-2개] | 채택/기각 |

> 옵션이 1개만 남은 경우, 다른 대안이 기각된 이유를 명시적으로 기술한다.
```

### Plan 본문 (10개 섹션)

품질 레퍼런스: Phase 1 Agent C가 발견한 저장소 내 기존 Plan (없으면 아래 템플릿 자체를 기준으로 삼는다)

**1. 기술 환경 요약** — 사용 스택, 기존 시스템 재활용 영역, MVP 범위 결정
**2. 전체 아키텍처** — ASCII 다이어그램 필수, 설계 핵심 원칙, **기각된 설계 vs 채택 설계 테이블 필수** (최소 2개 대안), 상세 흐름
**3. 데이터 모델 설계** — 신규/확장 테이블, 상태 전이 다이어그램, 상태 판별 로직
**4. API 설계** — 신규 엔드포인트, Request/Response 예시, 기존 엔드포인트 확장
**5. 프론트엔드 설계** — 컴포넌트 구조, 상태 관리, 주요 컴포넌트 변경 (해당 시)
**6. Implementation Tasks** — 고유 ID, 대상 레포지토리, 코드 스니펫 필수
**7. Task Dependencies & Execution Order** — 의존성 그래프, 실행 순서
**8. Cross-Repository Impact** — API Contract Changes, Data Schema Changes, 배포 순서
**9. Risk & Mitigation** — 리스크, 영향도, 발생 가능성, 대응 방안
**10. AC Coverage Matrix** — 모든 P0/P1 기능 요구사항이 최소 1개 Task에 매핑

> 각 섹션의 상세 템플릿은 품질 레퍼런스 Plan을 기준으로 작성한다. 최소 템플릿 수준이 아닌 팀 산출물 수준을 준수한다.

---

## Phase 3: Reviewer — Architect 리뷰 (순차 필수)

Plan 초안 작성 완료 후, **plan-architect agent** (subagent_type="plan-architect", model=opus)를 실행한다.

```
.claude/agents/plan-architect.md 정의를 따라 다음 Plan을 리뷰하라.

[Plan 초안 전문 또는 파일 경로]
[PRD 요약]
[Research 요약]

판정: APPROVE | ITERATE | REJECT
```

> **중요**: Architect 완료를 반드시 기다린 후 Phase 4로 진행한다. Architect의 피드백 없이 Critic이 평가하면 아키텍처 결함이 누락된다.

---

## Phase 4: Reviewer — Critic 평가 (Phase 3 완료 후)

Architect 리뷰 완료 후, **plan-critic agent** (subagent_type="plan-critic", model=opus)를 실행한다.

```
.claude/agents/plan-critic.md 정의를 따라 다음 Plan과 Architect 리뷰 결과를 평가하라.

[Plan (최신 버전)]
[Architect Review 결과]
[PRD 요약]

판정: APPROVE | ITERATE | REJECT
```

---

## Phase 5: 합의 루프 (최대 3회)

Critic 판정이 `APPROVE`가 아닌 경우:

1. Architect + Critic 피드백을 수집
2. Planner(본인)가 Plan을 수정
3. Phase 3 (Architect 리뷰) 재실행
4. Phase 4 (Critic 평가) 재실행
5. `APPROVE` 또는 3회 반복 시 종료

3회 반복 후에도 `APPROVE`가 아니면, 최선 버전을 사용자에게 제시하고 미해결 피드백을 명시한다.

---

## Phase 6: 최종 출력

합의 완료 후 Plan 파일을 저장하고 사용자에게 제시한다.

```markdown
## Plan 합의 완료

### 합의 과정 요약

| 라운드 | Architect 판정 | Critic 판정 | 주요 변경 |
|--------|---------------|-------------|----------|
| 1 | [판정] | [판정] | [변경 요약] |

### 다음 단계:
1. **설계 변경**: "기각된 대안 [X]를 재검토해줘"
2. **Task 분리/병합**: "TASK-01을 두 개로 나눠줘"
3. **리스크 추가**: "[리스크] 추가해줘"
4. **코드 스니펫 상세화**: "TASK-02 코드 더 구체적으로"
5. **플랫폼 엔지니어 리뷰**: `/review-plan` 실행
```

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| Research 파일 없음 | Plan 작성 중단, `/create-research` 안내 |
| 품질 레퍼런스 읽기 실패 | 일반 기준으로 작성, 레퍼런스 미참조 명시 |
| Architect agent 실패 | 1회 재시도, 재실패 시 Critic만으로 진행 (리뷰 1단계 축소 명시) |
| Critic agent 실패 | 1회 재시도, 재실패 시 Architect 결과만으로 진행 |
| 3회 루프 후 APPROVE 불가 | 최선 버전 제시 + 미해결 피드백 목록 명시 |

## Quick Checklist

- [ ] Phase 1: 3개 Agent 병렬 컨텍스트 수집 완료
- [ ] RALPLAN-DR 요약 포함 (설계 원칙, 의사결정 드라이버, 실행 가능한 옵션)
- [ ] 모든 10개 섹션 작성됨 (해당 없는 섹션은 "없음" 명시)
- [ ] ASCII 아키텍처 다이어그램 포함됨
- [ ] 기각된 설계 vs 채택 설계 테이블 포함됨 (최소 2개 대안)
- [ ] 데이터 모델 변경사항 명시됨
- [ ] API 설계 with Request/Response 예시 포함됨
- [ ] 모든 Task에 고유 ID, Repository, 코드 스니펫 포함됨
- [ ] Task 간 의존성 분석 완료
- [ ] 모든 P0/P1 기능 요구사항이 최소 1개 Task에 매핑됨
- [ ] Cross-repo 영향 범위 명시됨
- [ ] 배포 순서 정의됨
- [ ] Architect APPROVE 획득
- [ ] Critic APPROVE 획득

## 파일 저장

PRD 파일과 같은 디렉토리에 `plan.md` 또는 `PLAN.md`로 저장한다.

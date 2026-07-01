# Plan 기술 리뷰 (Fan-out/Fan-in Mode)

너는 시니어 플랫폼 엔지니어이며, 개발자가 작성한 Implementation Plan의 기술적 타당성을 리뷰한다. **Fan-out/Fan-in 패턴**으로 3개 리뷰 관점을 병렬 수행하고, 교차 종합하여 최종 판정을 내린다.

**아키텍처 패턴**: Fan-out/Fan-in (서브 에이전트 모드)

## 입력

$ARGUMENTS 로 Plan 파일 경로를 받는다. 입력이 없으면 현재 디렉토리에서 `plan.md` 또는 `PLAN.md`를 찾는다.

---

## Phase 1: 병렬 컨텍스트 로딩

**반드시 모든 파일을 먼저 읽는다** (본인이 직접):

1. **Plan 파일** (리뷰 대상)
2. **PRD 파일** (같은 디렉토리의 `prd.md` 또는 `PRD.md`)
3. **Research 파일** (같은 디렉토리의 `research.md`)
4. **environment.json** (레포지토리 이름, 타입, 역할 확인)
5. **팀/모듈 AGENTS.md** (있으면 레포 범위 제약 확인)

---

## Phase 2: Fan-out — 3개 리뷰 관점 병렬 실행

**3개 Agent를 동시에 실행**한다. 반드시 하나의 메시지에서 병렬 발행한다. 각 Agent에게 Plan, PRD, Research 내용을 전달한다.

### Agent A: 아키텍처 & 설계 리뷰 (model=sonnet)

```
Implementation Plan의 아키텍처와 설계를 리뷰하라.

리뷰 차원:
1. **AC Coverage 완전성** -- PRD의 모든 P0/P1 기능이 Task에 매핑되어 있는가?
   - 모든 P0 기능이 최소 1개 TASK에 매핑됨
   - Out-of-Scope 항목이 TASK에 포함되지 않음
   - 비기능 요구사항(성능/보안/정합성)이 반영됨

2. **아키텍처 적절성** -- 설계가 기존 시스템과 일관되고 불필요한 복잡도가 없는가?
   - 기존 시스템 재활용이 적절함 (Research 결과 기반)
   - 기각된 설계 대안이 공정하게 평가됨 (straw man 아닌가?)
   - 채택된 설계의 근거가 명확함
   - ASCII 아키텍처 다이어그램이 핵심 플로우를 정확히 표현함
   - RALPLAN-DR이 있으면 설계 원칙-옵션 일관성 확인

3. **Task 순서 타당성** -- Task 의존성 그래프가 올바르고 실행 순서가 효율적인가?
   - 순환 참조 없음
   - 독립 Task가 병렬화 가능으로 표시됨
   - 데이터/스키마 -> API -> UI 순서 지켜짐

각 차원별로 APPROVE / REQUEST_CHANGES / NEEDS_DISCUSSION 판정.
구체적 피드백에 Plan의 섹션 번호, TASK ID를 반드시 참조하라.
```

### Agent B: 데이터 & API 리뷰 (model=sonnet)

```
Implementation Plan의 데이터 모델과 API 설계를 리뷰하라.

리뷰 차원:
1. **데이터 모델 건전성** -- 스키마 변경이 안전하고 마이그레이션 전략이 적절한가?
   - 기존 테이블 확장 시 backward compatibility 유지
   - NOT NULL 컬럼 추가 시 default 값 또는 마이그레이션 전략
   - 인덱스 추가가 성능에 미치는 영향
   - 상태 전이 다이어그램이 모든 edge case 커버
   - 데이터 정합성 제약(UNIQUE, FK)이 비즈니스 규칙과 일치

2. **API 설계 리뷰** -- API 설계가 기존 패턴과 일관되고 Breaking Change 관리가 적절한가?
   - 기존 API 네이밍 컨벤션과 일관됨
   - Request/Response 형식이 기존 패턴을 따름
   - Breaking Change 시 버전 관리 전략
   - 에러 응답 형식이 기존 패턴과 일관됨
   - 인증/인가 요구사항 반영됨

3. **경계면 정합성** -- API 응답과 프론트 타입, DB 스키마 간 일관성
   - API 응답 shape과 프론트 타입 정의 일치 여부
   - snake_case ↔ camelCase 변환 일관성
   - 상태 전이 맵과 실제 업데이트 코드 일치 여부

각 차원별로 APPROVE / REQUEST_CHANGES / NEEDS_DISCUSSION 판정.
```

### Agent C: Cross-Repo & 리스크 리뷰 (model=sonnet)

```
Implementation Plan의 Cross-Repository 영향과 리스크를 리뷰하라.

리뷰 차원:
1. **Cross-Repo 영향 정확성** -- 영향받는 레포가 정확히 식별되고 배포 순서가 올바른가?
   - 영향받는 모든 레포지토리 식별됨
   - API Contract 변경 시 Provider -> Consumer 순서
   - Breaking Change 여부 정확히 표시됨
   - 배포 순서가 의존성 반영
   - 레포 범위를 벗어나는 변경이 없음

2. **리스크 식별** -- 중요 리스크가 누락되지 않고 대응 방안이 현실적인가?
   - 데이터 마이그레이션 리스크 (해당 시)
   - 동시성/경쟁 조건 리스크 (해당 시)
   - 외부 시스템 의존성 리스크 (해당 시)
   - 롤백 전략 명시됨
   - 각 리스크의 Mitigation이 구체적이고 실행 가능

각 차원별로 APPROVE / REQUEST_CHANGES / NEEDS_DISCUSSION 판정.
```

---

## Phase 3: Fan-in — 교차 종합 및 최종 판정

3개 Agent 결과를 수집한 후, 본인(메인 에이전트)이 종합 판정을 내린다.

### 3-1. Agent 간 교차 확인

```
- Agent A와 B의 결과에서 상충하는 판단이 있으면 직접 Plan을 읽어 확인
- Agent 간 중복 지적은 통합
- 한 Agent만 발견한 심각한 이슈는 다른 관점에서 재확인
```

### 3-2. 최종 리뷰 보고서 생성

```markdown
# Plan 기술 리뷰

**Plan**: [plan 파일 경로]
**PRD**: [prd 파일 경로]
**리뷰 일시**: [날짜]

## 종합 판정

| 차원 | 판정 | 요약 |
|------|------|------|
| AC Coverage | OK/CHANGES/DISCUSS | [한줄 요약] |
| 아키텍처 | OK/CHANGES/DISCUSS | [한줄 요약] |
| 데이터 모델 | OK/CHANGES/DISCUSS | [한줄 요약] |
| API 설계 | OK/CHANGES/DISCUSS | [한줄 요약] |
| 경계면 정합성 | OK/CHANGES/DISCUSS | [한줄 요약] |
| Cross-Repo 영향 | OK/CHANGES/DISCUSS | [한줄 요약] |
| 리스크 식별 | OK/CHANGES/DISCUSS | [한줄 요약] |
| Task 순서 | OK/CHANGES/DISCUSS | [한줄 요약] |

**최종 판정: APPROVE / REQUEST_CHANGES / NEEDS_DISCUSSION**

## 상세 피드백

### [차원명] -- [판정]

**발견 사항**:
- TASK-XX: [구체적 피드백]
- 섹션 N: [구체적 피드백]

**제안**:
- [구체적이고 실행 가능한 제안]

---

## 전체 의견

[Plan에 대한 종합적인 의견 3-5문장]
[리뷰 통과를 위해 반드시 수정해야 할 사항 나열]
```

## 판정 기준

| 판정 | 의미 | 다음 단계 |
|------|------|----------|
| APPROVE | 해당 차원에서 문제 없음 | 진행 가능 |
| REQUEST_CHANGES | 수정 필요한 문제 발견 | 수정 후 재리뷰 |
| NEEDS_DISCUSSION | 팀 논의가 필요한 사항 | 기술 미팅에서 논의 |

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| Review Agent 1개 실패 | 1회 재시도, 재실패 시 해당 관점 누락 명시하고 나머지로 진행 |
| Review Agent 과반 실패 | 사용자에게 알리고 단일 에이전트 순차 리뷰로 폴백 |
| PRD 파일 없음 | AC Coverage 검증 불가 명시, 나머지 차원만 리뷰 |
| Research 파일 없음 | 기존 시스템 재활용 적절성 검증 불가 명시 |

## 리뷰 원칙

- **증거 기반**: 모든 피드백에 Plan/PRD의 구체적 섹션/TASK 번호 참조
- **건설적**: 문제만 지적하지 말고 구체적 대안 제시
- **비례적**: 사소한 문구 수정이 아닌 구조적/기술적 이슈에 집중
- **실용적**: 이상적 설계가 아닌 현실적 제약 내에서의 최선에 집중
- **양쪽 동시 읽기**: 경계면 검증은 반드시 양쪽 코드를 동시에 비교

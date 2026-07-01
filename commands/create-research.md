# PRD 기반 코드베이스 Research (Fan-out/Fan-in Mode)

너는 PRD의 요구사항을 코드베이스에서 분석하여 구현 현황을 파악하는 전문 리서처다. **Fan-out/Fan-in 패턴**으로 요구사항별 리서치를 병렬 수행하고, **교차 검증**으로 경계면 불일치를 탐지한다.

**아키텍처 패턴**: Fan-out/Fan-in (서브 에이전트 모드)
**Agent 정의**: `.claude/agents/codebase-researcher.md`

## 입력

$ARGUMENTS 로 PRD 파일 경로를 받는다. 입력이 없으면 현재 디렉토리에서 `prd.md` 또는 `PRD.md`를 찾는다.

## 핵심 원칙

- **현재 상태만 문서화**: 개선 제안, 리팩토링 권고, 버그 지적 금지
- **증거 기반**: 모든 주장에 `file:line` 참조 필수
- **객관적 서술**: 사실만 기록, 의견 없음

---

## Phase 0+1: 병렬 컨텍스트 로딩

**2개 Agent를 동시에 실행**한다. 반드시 하나의 메시지에서 병렬 발행한다.

### Agent A: 환경 및 제약 로드 (Explore agent)

```
프로젝트 환경과 레포 제약을 분석하라:
1. 프로젝트 루트의 environment.json 읽기 -- 레포지토리 이름, 타입, 역할 매핑 추출
2. 팀/모듈 AGENTS.md (있으면) -- 레포 범위 제약 확인
3. 출력: REPO_LIST (이름, 타입, 역할), SCOPE 제약
```

### Agent B: PRD 파싱 및 요구사항 추출 (Explore agent)

```
PRD 파일을 전체 읽고 다음을 추출하라:
1. 모든 User Story (US-XX) 목록
2. 각 User Story에 연결된 기능 요구사항 (P0/P1 우선순위 포함)
3. 비기능 요구사항, 기술 고려사항
4. 각 요구사항별 검색 키워드 (최소 5개)

키워드 추출 규칙:
- 도메인 용어 (예: 검색, 주문, 결제, 알림)
- 기술 스택 컴포넌트/서비스명 (예: SearchService, OrderRepository)
- UI 요소 화면/컴포넌트명 (예: SearchInput, OrderCard)
- API 엔드포인트 경로 패턴 (예: /api/search, /orders)
- 한글+영문 변형 모두 포함 (예: "주문내역" + "order-history" + "orderHistory")

출력 형식:
REQ-US-01: [제목]
  > [원문 User Story]
  관련 기능: P0: [기능명], P1: [기능명]
  Keywords: [keyword1, keyword2, keyword3, keyword4, keyword5]
```

Research가 없으면 다음 Phase로 진행한다 (Research는 자체가 Research이므로).

---

## Phase 2: Fan-out — 병렬 요구사항 리서치

Phase 0+1 결과를 종합한 후, **codebase-researcher agent** (subagent_type="codebase-researcher", model=opus)를 요구사항 그룹별로 병렬 실행한다.

### 그룹핑 규칙

| 요구사항 수 | 그룹핑 전략 | 병렬 Agent 수 |
|------------|-----------|-------------|
| 1-3개 | 각 요구사항마다 1개 Agent | 최대 3개 |
| 4-6개 | 관련도 높은 것끼리 2개씩 | 최대 3개 |
| 7개 이상 | 관련도 기반 3그룹으로 | 3개 |

> 병렬 Agent 수는 최대 3개로 제한한다. 3명의 집중된 에이전트가 5명의 산만한 에이전트보다 낫다 (harness 팀 크기 가이드라인).

### 각 Research Agent 프롬프트

```
.claude/agents/codebase-researcher.md 정의를 따라 다음 요구사항을 리서치하라.

## 검색 범위
REPOS: [Phase 0에서 추출한 레포지토리 목록 (이름, 타입)]
SCOPE: [AGENTS.md 제약]

## 담당 요구사항
[REQ-US-XX 요구사항 내용과 키워드]

## 리서치 절차

Step 1: 넓은 검색 (Grep + Glob 병렬)
- Grep: 각 키워드 변형으로 코드 내용 검색
- Glob: 파일명 패턴 매칭
- 정규식으로 유사어 동시 검색 (예: (order|purchase|주문))

Step 2: 심층 분석
- 발견된 파일 읽기, 구현 흐름 추적
- import 경로를 따라가 관련 파일 탐색
- 테스트 파일 확인 (의도된 동작 파악)

Step 3: 갭 분석
- 기능 요구사항 대비 구현 현황 비교
- 누락된 부분 식별 (솔루션 제안 없이 사실만)

## 출력 형식

### REQ-US-XX: [제목]
#### Status: [Implemented | Partial | Not Found]
#### Files Found
| Type | Path | Relevance |
#### Implementation Summary
[2-3문장]
#### Coverage
- [v] [구현된 부분]
- [x] [누락된 부분]
#### Key Code References
- `path/file.ts:45` -- [참조 이유]
```

---

## Phase 2.5: Fan-out — 병렬 심층 분석

Phase 2 결과에서 **Implemented 또는 Partial 상태인 요구사항**에 대해서만 심층 분석 Agent를 병렬 실행한다 (subagent_type="codebase-researcher", model=opus).

### 심층 분석 Agent 프롬프트

```
다음 요구사항의 구현을 심층 분석하라.

## Entry Points
[Phase 2에서 발견한 Key Code References]

## 분석 절차
1. Entry Point 파일을 읽고 구현 로직 파악
2. import/호출 경로를 따라가며 데이터 흐름 추적
3. Frontend -> API -> Service -> DB 전체 흐름 매핑
4. 테스트 파일에서 의도된 동작 확인

## 출력 형식

### Data Flow
[User Action] -> [Frontend] -> [API] -> [Service] -> [Response]
(각 단계에 file:line 참조)

### Component Breakdown
#### [Component Name] (`path/to/file.tsx:lines`)
**Purpose**: [역할]
**Key Logic**: [코드 스니펫]
**Connections**: Imports, Calls
```

---

## Phase 3: Fan-in — 교차 검증 및 문서 종합

모든 병렬 Agent 결과를 수집한 후, 본인(메인 에이전트)이 다음을 수행한다.

### 3-1. 교차 검증

Agent 간 결과를 대조하여 불일치를 탐지한다:

```
1. 중복 발견된 파일/함수를 통합
2. Agent 간 상충되는 결과가 있으면 직접 파일을 읽어 확인
3. Not Found 판정된 요구사항에 대해 누락된 키워드 변형 1회 추가 검색
```

### 3-2. 경계면 불일치 탐지

여러 요구사항에 걸쳐 다음 경계면을 교차 비교한다. API와 프론트 코드가 각각 "올바르게" 구현되어 있어도 연결 지점에서 계약이 어긋나는 경우가 가장 빈번한 결함이다:

| 경계면 | 검증 방법 |
|--------|----------|
| API 응답 shape ↔ 프론트 훅 타입 | NextResponse.json() shape vs fetchJson<T> 타입 비교 |
| 파일 경로 ↔ 링크/라우터 경로 | page 파일 URL 패턴 vs href/router.push 값 대조 |
| 상태 전이 맵 ↔ 실제 status 업데이트 | 허용 전이 vs .update({status}) 코드 대조 |
| DB 스키마 필드 ↔ API 응답 필드 | snake_case ↔ camelCase 변환 일관성 |

> 경계면 검증은 반드시 **양쪽 코드를 동시에 열어** 비교한다. 한쪽만 읽으면 불일치를 놓친다.

### 3-3. research.md 생성

```markdown
---
date: [날짜]
prd_source: "[prd 파일 경로]"
topic: "PRD Implementation Analysis"
status: complete
---

# PRD Implementation Analysis

**PRD**: [prd 파일 경로]
**Date**: [날짜]

## Summary

| Total | Implemented | Partial | Not Found |
|-------|-------------|---------|-----------|
| [n] | [n] | [n] | [n] |

## Requirements Analysis

### REQ-US-01: [Title]
> [Original User Story]
#### Research Result
[Phase 2 결과]
#### Implementation Analysis
[Phase 2.5 결과 -- Implemented/Partial인 경우만]
---

## Cross-Cutting Patterns
[여러 요구사항에서 공통으로 발견된 패턴, 공유 서비스, 공통 테이블]

## Boundary Analysis
[경계면 불일치 탐지 결과 -- 발견된 불일치 또는 "불일치 없음"]

## All Code References
[전체 file:line 참조 목록 통합 -- 중복 제거]
```

---

## Phase 4: 결과 제시

요약을 제시하고 후속 질문을 받는다:
- 추가 검색이 필요한 영역
- 불명확한 구현 부분 확인
- 경계면 불일치가 발견되었으면 Plan 작성 시 주의 사항 안내

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| Agent 1개 실패 | 1회 재시도. 재실패 시 해당 결과 없이 진행, 보고서에 "미수집" 명시 |
| Agent 과반 실패 | 사용자에게 알리고 진행 여부 확인 |
| 대상 레포지토리 검색 결과 없음 | 키워드 변형 후 재검색, 재실패 시 보고서에 "미수집" 명시 |
| 검색 결과 과다 (100건+) | 파일 타입/경로로 필터링 후 상위 30건만 분석 |
| Not Found 의심 | 키워드 변형 3회 이상 시도 후 최종 판정 |

## Status 기준

| Status | 의미 |
|--------|------|
| Implemented | 요구사항의 모든 측면이 코드에 구현되어 있음 |
| Partial | 일부 구현, 나머지 누락 |
| Not Found | 해당 요구사항에 대한 구현을 찾지 못함 |

## 병렬 실행 요약

| Phase | Agent 수 | 패턴 | 목적 |
|-------|----------|------|------|
| 0+1 | 2 | Fan-out | 환경 로드 + PRD 파싱 |
| 2 | 최대 3 | Fan-out | 요구사항별 코드 리서치 |
| 2.5 | 최대 3 | Fan-out | 구현된 요구사항 심층 분석 |
| 3 | 1 (본인) | Fan-in | 교차 검증 + 경계면 탐지 + 문서 종합 |

## 하지 말 것

- 읽지 않은 코드에 대해 추측하지 말 것
- 테스트/설정 파일 검색을 건너뛰지 말 것
- 개선 방안, 리팩토링, 대안을 제시하지 말 것
- 코드 품질이나 아키텍처를 평가하지 말 것
- "Not Found" 선언 전에 키워드 변형 3회 이상 시도할 것
- Agent 3개 초과 병렬 실행하지 말 것

## 파일 저장

PRD 파일과 같은 디렉토리에 `research.md`로 저장한다.

# TC 시트 생성 (Fan-out/Fan-in Mode)

너는 QA 전문가이며, PRD와 Plan을 분석하여 구조화된 테스트 케이스(TC) CSV를 생성한다. **Fan-out/Fan-in 패턴**으로 기능 영역별 TC를 병렬 생성하고, P0/P1 커버리지 100% 검증 후 CSV로 출력한다.

**아키텍처 패턴**: Fan-out/Fan-in (서브 에이전트 모드)
**Agent 정의**: `.claude/agents/qa-tc-generator.md`
**입력 우선순위**: Plan (구현 Task 기반) + PRD (수용 기준·우선순위 태깅) — 양쪽 모두 참조

## 입력

$ARGUMENTS 로 파일 경로를 받는다. 입력 형태:
- 경로 미입력: 현재 디렉토리에서 `plan.md`/`PLAN.md` + `prd.md`/`PRD.md` 자동 탐색
- 단일 경로: PRD 또는 Plan 파일 경로 — 나머지 하나는 같은 디렉토리에서 자동 탐색
- 두 경로: `{prd경로} {plan경로}` 순서로 명시
- **Figma URL**(인자 또는 PRD/PLAN 내 링크): 있으면 Phase 3.5에서 화면 대조에 사용 (프론트엔드 연동)

---

## Design Rationale (RALPLAN-DR)

### 설계 원칙

1. **PRD + Plan 이중 참조**: PRD는 수용 기준·우선순위를, Plan은 구현 Task·API·데이터 모델을 제공한다. 두 문서를 함께 참조해야 TC가 구현 수준의 경계값을 정확히 반영한다.
2. **Fan-out by Feature Area**: 기능 영역별 독립 병렬 생성으로 커버리지 누락을 최소화한다.
3. **CSV Output**: xlsx 대신 CSV로 Git 추적 가능성과 도구 범용성을 확보한다.
4. **블랙박스 우선 + 비개발자 가독성**: `TC_전체.csv`에 들어가는 QA TC는 **PM·기획·비개발 QA가 도구 없이 화면·행동만으로 따라갈 수 있게** 작성한다. Test Steps/Expected는 눈에 보이는 결과 중심으로 쓰고, DB·트랜잭션·내부 상태 같은 개발 검증은 넣지 않는다.
5. **QA / 개발 테스트 분리** ⭐: 모든 후보 TC를 **블랙박스(QA)** 와 **화이트박스(개발)** 로 분류한다. 블랙박스로 안정 재현이 불가능한 것(동시성·락·트랜잭션 원자성·PG 멱등키·시계 경계 등)은 QA 시트가 아니라 `개발테스트_체크리스트.md`로 분리한다.
6. **완전성 우선 — 빠짐없이 한 번에** ⭐: 첫 산출에서 생애주기 전 구간(진입→가입→전환→사용/혜택→**갱신·재시도·자동해지**→해지→환불→재가입/역전환)과 단면(멱등·동시성·경계값·무회귀)을 빠짐없이 훑는다. 특정 기능만 보고 후반 생애주기를 빠뜨리는 것이 가장 흔한 누락이다.
7. **Coverage Gate**: P0 기능 누락 TC는 Phase 3에서 반드시 재생성 후 완료한다.

### 핵심 의사결정 드라이버

1. **Plan Task → TC 매핑**: Plan의 TASK-XX가 있으면 TC와 매핑하여 구현-검증 추적 가능성 확보.
2. **PRD AC → Expected Result**: PRD 수용 기준(AC-XX, Given/When/Then)을 Expected Result에 직접 반영.
3. **Plan 없이 PRD만 있는 경우도 지원**: PRD 단독 입력 시 기능 요구사항 기반으로 TC 생성.

### 실행 가능한 옵션

| 옵션 | 설명 | 장점 | 단점 | 결정 |
|------|------|------|------|------|
| A: PRD만 입력 | PRD 기능 요구사항 기반 TC | 초기 단계 활용 가능 | 구현 경계값 반영 불가 | 폴백 |
| B: PRD + Plan (채택) | PRD 수용 기준 + Plan Task 병합 참조 | 구현-검증 일치, 경계값 정확 | Plan 미완성 시 제한 | **채택** |

---

## Phase 0: 입력 검증 (본인 직접 수행)

1. **파일 탐색**: PRD + Plan 파일 위치 확정
   - 두 파일 모두 있음 → Phase 1 (Full Mode)
   - PRD만 있음 → Phase 1 (PRD-Only Mode, Plan 참조 없음 명시)
   - Plan만 있음 → Phase 1 (Plan-Only Mode, PRD 수용 기준 없음 명시)
   - 아무것도 없음 → **즉시 중단** + `/create-prd`, `/create-plan` 안내
2. PRD가 있으면: P0/P1 기능 요구사항, User Story(US-XX) 확인
3. Plan이 있으면: TASK-XX 목록, API 설계, 데이터 모델 확인

---

## Phase 1: 문서 분석 → 기능 트리 추출 (본인 직접 수행)

### Full Mode (PRD + Plan 양쪽 보유)

1. **PRD에서 추출**:
   - User Story 목록 (US-XX ID)
   - P0/P1 기능 요구사항 + 수용 기준 (AC-XX, Given/When/Then)
   - Out-of-Scope 항목 (TC 제외 대상)

2. **Plan에서 추출**:
   - TASK-XX 목록 + 각 Task의 대상 기능
   - API 엔드포인트 + Request/Response 경계값
   - 데이터 모델 상태 전이 (상태값 경계 → Edge Case TC 도출)
   - RALPLAN-DR의 기각 설계 (TC 대상 제외 확인)

3. **PRD ↔ Plan 매핑 테이블 작성**:
   ```
   | US-XX / 기능 | TASK-XX | TC Depth 1 | 비고 |
   |-------------|---------|------------|------|
   ```

### PRD-Only / Plan-Only Mode

- 보유 문서 기반으로 기능 트리 추출
- 누락 문서로 인해 검증 불가한 항목 명시

### 완전성 스윕 (누락 방지 — 빠짐없이 한 번에) ⭐

기능 트리를 짜기 전, 아래 "생애주기 + 단면" 체크리스트를 PRD/Plan에 대조해 **누락 영역이 없는지 먼저 확인**한다. 가입/전환/해지 같은 눈에 띄는 기능만 보고 갱신·재시도 같은 생애주기 후반을 빠뜨리는 것이 가장 흔한 누락이다.

- **생애주기 전 구간**: 진입/노출 → 가입 → (전환/플랜 변경) → 사용/혜택 적용 → **갱신·재결제 → 재시도/grace/자동해지** → 해지 → 환불 → 재가입/역전환
- **상태 전이**: 데이터 모델의 모든 상태값 경계(ACTIVE/PENDING/SCHEDULED/CANCELLED/SUSPENDED/PAST_DUE 등)에서 진입·거부 케이스
- **단면(cross-cutting)**: 멱등(재시도 이중처리)·동시성(같은 주체 동시 요청)·경계값(금액·날짜·윤년·자정)·캐시 정합성·**무회귀(기존 기능 유지)**·권한/소유권
- **비기능 정합성**: 원자성/트랜잭션·금액 SSOT·규제 고지

PRD/Plan에 근거가 있는데 트리에 없으면 [MISSING] 후보로 적고 반드시 분류한다. 이 스윕 결과를 Phase 3 [LIFECYCLE] 게이트에서 재검증한다.

### TC 검증 유형 분류 (블랙박스 QA vs 개발 테스트) ⭐

각 후보 TC를 **검증 방식**으로 분류한다. 이 분류가 출력 파일을 가른다.

| 유형 | 정의 | 출력 위치 |
|------|------|----------|
| **QA (블랙박스)** | 실제 계정·화면·API 응답으로 결과를 관찰 가능. 셋업은 QA 스킬(백데이팅·빌링키 주입 등)로 가능 | `TC_전체.csv` |
| **개발 (화이트박스)** | fault injection(@Tx 롤백·publish 실패·타임아웃 주입)·race 시뮬레이션·내부 상태(락/EM/캐시)·PG 멱등키 헤더 전파·시계 경계 단위테스트 등 **블랙박스로 안정 재현 불가** | `개발테스트_체크리스트.md` |
| **협업** | 셋업/내부 검증은 개발, 결과 관찰은 QA (예: 결제 실패 주입 → 자동해지 결과 확인) | `TC_전체.csv`에 두되 비고/【담당자·도구】에 "개발 협조" 표기 |

**판정 질문**: "이 케이스를 수동으로 안정적으로 재현할 수 있는가?" → No 이면 개발.
- 개발 신호: 동시 요청 race, 락 TTL/토큰 재검증, EM 격리, 멱등키 헤더 전파, 타임아웃 후 orphan 복구, 트랜잭션 원자성/롤백, 캐시 무효화 내부 상태, 환불 공식·날짜 경계 단위테스트
- QA 신호: 화면 문구·금액·CTA 노출, 가입/전환/해지/갱신 결과, 상태 전이 결과, 에러 안내 메시지, 무회귀 화면 확인

### 기능 그룹화 (공통)

최대 3개 그룹으로 분류:
- 그룹 기준: **사용자가 보는 화면 단위**(Depth 1) — 동일 화면/User Journey끼리 묶는다. API·내부 동작 기준 묶음 금지
- 각 그룹에 FLOW 번호 범위, TC ID 시작 번호 배정
  - 그룹 A: TC-01~, 그룹 B: TC-21~, 그룹 C: TC-41~

기능 트리 출력:
```
기능 트리 (Full Mode):
그룹 A (FLOW 1-N, TC-01~): [기능 영역명]
  - [Depth 1] / [Depth 2] — US-XX (P0), TASK-XX
그룹 B (FLOW N+1-M, TC-21~): [기능 영역명]
  - [Depth 1] / [Depth 2] — US-XX (P1), TASK-XX
```

---

## Phase 2: Fan-out — 기능 그룹별 TC 초안 생성

그룹 수에 따라 **최대 3개 Agent를 동시에 실행**한다. 반드시 하나의 메시지에서 병렬 발행한다.
**subagent_type**: `qa-tc-generator` (`.claude/agents/qa-tc-generator.md` 참조)
**model**: sonnet

각 Agent에게 전달:
- PRD 해당 그룹 기능 요구사항 + AC (수용 기준 전문)
- Plan 해당 그룹 TASK 목록 + API/데이터 모델 경계값
- 담당 기능 그룹명, FLOW 번호 범위, TC ID 시작 번호
- Full/PRD-Only/Plan-Only 모드 명시
- **검증 유형 분류 지시**: 블랙박스(QA) TC는 화면·행동 중심으로, 화이트박스(개발) 항목은 별도 표기하여 반환

**⚠️ 출력 계약 (산출물 유실 방지)**: 각 Agent는 TC 행 목록을 **JSON 배열로 `/tmp/tc_group_{A|B|C}.json` 파일에 Write 도구로 저장**하고, 최종 메시지로는 "saved N TCs to {경로}"만 보고한다. (Agent 최종 메시지에 TC 본문을 그대로 싣게 하면 회수가 누락될 수 있다 — 반드시 파일로 저장.) 본인은 Phase 3에서 이 파일들을 읽어 통합한다.

### TC 컬럼 규칙 (각 Agent 공통 준수)

| 컬럼 | 규칙 |
|------|------|
| **FLOW** | `FLOW 1`, `FLOW 2` ... — 동일 계정+Device로 연속 실행 가능한 묶음 |
| **ID** | `TC-01`, `TC-02` ... FLOW 안에서 연속 순번 유지 |
| **Depth 1** | **사용자가 보는 화면 기준** 대분류 ⭐ — API·내부 동작(예: "Phase0 사전 거부", "outbox 재처리", "plan-매칭 가드")이 아니라 **유저가 실제 도달하는 화면**으로 적는다. (예: `멤버십 가입 화면`, `상품·장바구니 화면`, `멤버십 관리 - 연간 전환`, `멤버십 관리 - 중도 해지`, `멤버십 관리 - 상태/갱신`) |
| **Depth 2** | 그 화면 내 **영역·상황** (예: `플랜 선택 토글·가격 표시`, `전환 미리보기(구매이력 없음)`, `환불 금액(기본 계산)`, `해지 예약 중 다시시작 버튼 비노출`). 경계값/조건은 괄호로 명시. 응답코드·내부 함수명 금지 |
| **Test Case Name** | 한 문장 시나리오 요약 — **사용자 관점**, 개발 용어·응답코드 단독 금지 |
| **전제 계정/Device** | 계정 레이블 + Device 그룹 (예: `계정B / Device Y(Treatment)`) |
| **Test Steps** | **한 문장**(누가 + 무엇을 + 행동). `[전제조건]` 블록·다단계 `→` 금지 — 상태는 문장 주어로 접는다(예: "구매이력 없는 월간 멤버가 연간으로 전환"). 상세 셋업(백데이팅 등)은 `TC_계정구성.csv`가 담당. 단, 경계값 여러 개를 한 번에 확인하는 TC만 예외적으로 짧은 `①②③` 허용 |
| **Expected Result** | **눈에 보이는 결과 한 줄**(길어도 1~2줄). 표시 금액·문구·버튼 상태만. `【담당자·도구】`·DB·API·HTTP 코드·JSON 예시·`(개발 확인)` 금지. 경계값 TC만 `①②③` |
| **우선순위** | Critical / High / Medium / Low |
| **PC, MOBILE(iOS), MOBILE(Android)** | 빈칸 (QA 실행 시 기록용) |

### 블랙박스 TC 작성 규칙 (비개발자 가독성) ⭐

`TC_전체.csv`의 QA TC는 **PM·기획·비개발 QA가 도구 없이 한눈에 따라갈 수 있어야** 한다. 아래 골드 스탠다드 수준의 **극도로 간결한** 표현을 목표로 한다.

**목표 스타일 (실제 팀 TC 레퍼런스):**

| Test Steps | Expected Result |
|---|---|
| A그룹 유저가 4만원 상온 상품 담고 결제 | 배송비 3000원 부과 및 결제 |
| 미로그인 유저가 상온 상품 담기 | 기존 담기 동작 정상 노출 (로그인 유도 등) |
| 구매이력 없는 월간 멤버가 연간으로 전환 | 월간 즉시 해지 + 2,990원 환불 + 연간 멤버십 시작 |
| 사용 5개월·구매이력 있는 연간 멤버가 해지 | 환불 예상액 15,040원 표시(사용한 만큼 차감) |

**필수 규칙:**
1. **`[전제조건]` 블록 금지** — 계정 상태(비멤버십/월간/연간·구매이력 유무·사용 N개월·금액)는 Test Steps 문장의 **주어**로 접는다. 백데이팅 등 상세 셋업은 `TC_계정구성.csv`가 담당.
2. **Test Steps = 한 문장** — 다단계 `→` 나열 금지. 경계값 여러 개를 한 번에 확인하는 TC만 예외적으로 짧은 `①②③`.
3. **Expected = 눈에 보이는 결과 한 줄**(길어도 2줄) — 표시 금액·안내 문구·버튼 노출/비노출만.
4. **금지 항목**: `【담당자·도구】` 블록, DB row·트랜잭션·EM/락/캐시·내부 상태, API 경로·응답 필드명, HTTP 코드 숫자, `(※ JSON 예: ...)`, `(개발 확인 필요)` 라인. 이 모든 개발 검증은 `개발테스트_체크리스트.md`로 보낸다.
5. **기술 식별자 → 사람 말**: 내부 코드값(`ZMM_ANNUAL` 등)→"연간 멤버십", 상태값(`CANCELLATION_SCHEDULED`)→"해지 예약 상태" 등. 금액·날짜는 유지.
6. **Test Case Name**도 사용자 관점 짧은 문장(응답 코드·내부 용어 단독 금지).
7. **협업/개발 검증 TC**: 화면 관찰부만 평문으로 남기고, 백엔드 검증은 본문에서 빼서 개발 체크리스트로 보낸다(이미 분류된 `[개발 통합테스트]` TC는 "(상세는 개발 체크리스트)" 한 줄만).

### 시나리오 유형 (각 Depth 2마다 최소 1개 포함)

| 유형 | 도출 소스 |
|------|----------|
| Happy Path | PRD User Story 정상 플로우 |
| Edge Case | Plan API 경계값, 데이터 모델 상태 전이 |
| Error Case | PRD 비기능 요구사항, Plan 에러 응답 설계 |
| 권한/상태 전환 | PRD A/B 테스트, Plan 인증·인가 설계 |

### 병합 원칙

- 동일 계정 + 연속 실행 가능한 경계값 케이스 → Test Steps `①②③`으로 병합
- 서로 다른 계정이 필요한 케이스는 병합 금지
- Plan API 경계값이 연속이면 한 TC에서 값만 바꿔 연속 확인

---

## Phase 3: Fan-in — TC 통합 + 커버리지 검증

3개 Agent 결과 수집 후 **본인이** 통합한다.

### 3-1. TC 통합

1. 전체 TC를 그룹 A → B → C 순으로 합산
2. ID 연번 재확인, FLOW 번호 전체 기준 재조정
3. 중복 TC 탐지 + 병합

### 3-2. 커버리지 검증 체크리스트

```
□ PRD의 모든 P0 기능 → 최소 1개 Critical/High TC에 매핑됨
□ PRD의 모든 P1 기능 → 최소 1개 Medium TC에 매핑됨
□ (Full Mode) Plan의 모든 TASK-XX → 최소 1개 TC와 연결됨
□ 각 Depth 2마다 Happy Path + 최소 1개 Edge/Error Case 존재
□ [LIFECYCLE] 생애주기 누락: 갱신·재결제·재시도·자동해지·역전환 등 후반 구간이 빠지지 않음 (완전성 스윕 재대조)
□ [단면] 멱등(재시도 이중처리)·동시성·경계값(금액/날짜/윤년/자정)·무회귀 케이스 포함 (블랙박스 불가분은 개발로 분류)
□ [CLASS] 모든 TC가 QA / 개발 / 협업 중 하나로 분류됨 (미분류 0건)
□ [BLACKBOX] TC_전체.csv의 Test Steps/Expected가 비개발자 가독 — DB·트랜잭션·내부 상태 검증이 섞이지 않음
□ [VAGUE] 탐지: Test Steps가 모호한 TC
□ [WEAK] 탐지: Expected Result가 검증 불가능한 TC
□ [MISSING] 탐지: PRD 기능이지만 TC가 없는 항목
□ Out-of-Scope 기능이 TC에 포함되어 있지 않음
```

[MISSING]·[LIFECYCLE] 누락 또는 P0 커버리지 미달 → 보완 Agent 재실행 (최대 2회)
[CLASS] 미분류·[BLACKBOX] 위반 → 해당 TC를 재작성하거나 개발테스트 체크리스트로 이동
모두 통과 → Phase 4

### 3-3. 계정 구성 분석

전체 TC의 `전제 계정/Device` 컬럼을 파싱하여:
- 계정 유형 목록 도출
- 각 계정별 사용 FLOW, TC 수, 예상 테스트 횟수 집계

---

## Phase 3.5: Figma 화면 대조 (프론트엔드 연동) ⭐

**Figma 링크/MCP가 있으면 반드시 수행한다.** 블랙박스 TC의 Test Steps/Expected를 **실제 디자인 화면의 문구·금액·버튼·상태**로 보정해, 스펙 기반 추정이 아니라 디자인 정합 TC를 **1패스로** 산출한다. (이 단계가 없으면 화면 문구·금액이 추정값으로 남아 QA가 실물과 어긋난다.)

### 3.5-0. Figma 입력 확보
- PRD/PLAN에 Figma 링크가 있거나 사용자가 URL을 주면 사용. URL `figma.com/design/<fileKey>/...?node-id=<a>-<b>` → `fileKey`, `nodeId`(`a:b`로 변환).
- **MCP 우선순위**: 공식 Figma MCP 도구(get_metadata / get_screenshot). 미인증이면 사용자에게 MCP 인증 요청.
- Figma가 전혀 없으면 이 단계 스킵하고 Phase 5에서 "디자인 미대조(스펙 기반)" 명시.

### 3.5-1. 화면 인벤토리 (구조 파악)
- `get_metadata(fileKey, nodeId)`로 구조 취득. ⚠️ **출력이 거대하면(수십만 자) 전체 덤프 금지** — 파일로 저장된 결과를 **name 키워드로 검색**(예: 결제/토글/전환/해지/완료/예약/원가)해 화면 프레임 node id를 추린다.
- 화면(보통 폭 ~390/393, 높이 ~844+ 프레임)을 기능 영역에 매핑: 가입 화면 / 관리 화면 / 전환 / 해지 / 완료·모달 등.

### 3.5-2. TC ↔ 화면 3분류
각 블랙박스 TC를 분류한다:
- **(A) 화면 있음** → get_screenshot으로 대조 (3.5-3)
- **(B) 전용 화면 없음** → 안내 팝업/토스트·원장 검증 등 화면이 아닌 것 → 스펙 기반 유지
- **(C) 백엔드/계산/회귀** → 갱신 cron·환불 계산식·멱등·무회귀 등 화면 무관 → 스펙 기반 유지

### 3.5-3. 화면 캡처 → 실물 대조 (분류 A)
- `get_screenshot(fileKey, nodeId, maxDimension≈720)` → 반환 URL을 다운로드 → **Read로 이미지 확인**.
- ⚠️ **프레임 이름 ≠ 실제 내용**: 반드시 스크린샷을 눈으로 본다(프레임 이름만 보고 분류하면 오판 가능).
- 화면에서 **실제 라벨/문구/금액/취소선·배지/버튼 노출·비노출**을 읽어 Test Steps·Expected를 그 문구로 교체.
- 여러 상태(토글 on/off, case 분기, 성공/실패 모달)는 각 상태 화면을 캡처해 대응 TC에 매핑.

### 3.5-4. 디자인 ↔ 스펙 불일치 기록
대조 중 발견한 차이는 `TC_커버리지_갭.md`에 기록(임의 수정 금지, PO/디자인 판단 사안):
- **DSC**(디자인↔스펙 불일치): 카피·금액·할인율·표시 위치가 PRD와 다름 → PO 결정 필요
- **DSG**(화면 누락): 스펙엔 있는데 디자인에 화면이 없음 → 디자인 확인 필요
- 목업 placeholder 오류(예: 금액 오타)는 "무시 가능"으로 표기하되 기록은 남긴다.

> ⚠️ 자가 결론 금지: 불일치는 "확인 필요"로 남기고 사용자/PO 결정 후 반영. 단정적으로 TC를 바꾸지 않는다.

### 3.5-5. 결과 반영
- 분류 A의 TC는 디자인 문구로 보정한 최종본을 Phase 4 CSV에 반영.
- Phase 5 보고에 **대조 커버리지**(A 대조완료 N건 / B 화면없음 N건 / C 백엔드·계산 N건)와 DSC/DSG 목록을 포함.

---

## Phase 4: CSV 출력

**Python 코드를 직접 실행(Bash)하여** CSV 파일을 생성한다.

### 출력 파일 위치

PRD/Plan 파일과 동일한 디렉토리에 저장:
- `TC_전체.csv` (기능 영역 특정 시 `TC_{기능영역명}.csv`) — **블랙박스 QA TC**
- `TC_계정구성.csv`
- `개발테스트_체크리스트.md` — **화이트박스 개발 TC** (개발로 분류된 항목이 1건 이상일 때 생성)

### Python 코드 실행 규칙

인코딩: `utf-8-sig` (Excel 한글 깨짐 방지)

```python
import csv, os

tc_rows = [
    # {"type": "separator", "flow_label": "--- FLOW 1 ---"},
    # {"type": "tc", "flow": "FLOW 1", "id": "TC-01", "depth1": "대분류", "depth2": "중분류",
    #  "name": "시나리오 요약", "account": "계정A", "steps": "[전제조건]\n→ 액션1",
    #  "expected": "① 결과1", "priority": "Critical"},
]

headers = [
    "FLOW", "ID", "Depth 1", "Depth 2", "Test Case Name",
    "전제 계정/Device", "Test Steps", "Expected Result",
    "우선순위", "PC", "MOBILE(iOS)", "MOBILE(Android)"
]

output_dir = "{PRD_또는_PLAN_디렉토리_경로}"
tc_path = os.path.join(output_dir, "TC_전체.csv")

with open(tc_path, "w", newline="", encoding="utf-8-sig") as f:
    w = csv.writer(f)
    w.writerow(headers)
    for row in tc_rows:
        if row["type"] == "separator":
            w.writerow([row["flow_label"]] + [""] * 11)
        else:
            w.writerow([
                row.get("flow", ""), row.get("id", ""),
                row.get("depth1", ""), row.get("depth2", ""),
                row.get("name", ""), row.get("account", ""),
                row.get("steps", ""), row.get("expected", ""),
                row.get("priority", ""), "", "", ""
            ])

print(f"✓ TC 시트 생성: {tc_path}")
print(f"  총 TC: {sum(1 for r in tc_rows if r['type'] == 'tc')}개")
```

계정 구성 CSV:

```python
account_path = os.path.join(output_dir, "TC_계정구성.csv")
account_headers = ["구분", "계정/Device", "조건", "사용 FLOW", "예상 테스트 횟수", "비고"]

# Phase 3-3에서 집계한 계정별 데이터를 아래 형식으로 준비한다.
# account_rows = [
#     {"group": "테스터", "account": "계정A(비멤버십)", "condition": "멤버십 미가입",
#      "flows": "FLOW 1, FLOW 2", "count": 5, "note": ""},
# ]
account_rows = []  # Phase 3-3 분석 결과로 채운다

with open(account_path, "w", newline="", encoding="utf-8-sig") as f:
    w = csv.writer(f)
    w.writerow(account_headers)
    for r in account_rows:
        w.writerow([r["group"], r["account"], r["condition"],
                    r["flows"], r["count"], r.get("note", "")])
    w.writerow([])
    w.writerow(["합계", f"계정 {len(account_rows)}종", "", "",
                sum(r["count"] for r in account_rows), ""])

print(f"✓ 계정 구성 가이드: {account_path}")
```

### 개발테스트_체크리스트.md 출력 (화이트박스) ⭐

Phase 1에서 **"개발(화이트박스)"로 분류된 항목**을 별도 마크다운으로 저장한다(CSV 아님 — 코드 레벨 검증이라 표 형태 체크리스트가 적합). 개발 분류 항목이 0건이면 생성하지 않는다.

- 파일: `{동일 디렉토리}/개발테스트_체크리스트.md`
- 구성:
  - **통합 테스트** (fault injection·race): 동시성/락·EM 격리·멱등키 전파·타임아웃/orphan 복구·트랜잭션 원자성·자동환불·TOCTOU 등
  - **단위 테스트** (계산·경계): 금액 공식·사용량 산정·식별 로직·날짜 경계(윤년/말일/자정)
  - **내부 상태/배선**: 캐시 무효화·결제수단 선택·경로 분기·무회귀 보장
  - **미결 결정** (PO·개발 협의 필요): 스코프 경계 갭 등
  - **범위 밖** (타 팀/역할): 규제 문구(법무), 분석/dbt(data), Analytics 이벤트(FE)
- 각 항목: `검증 방법` + `근거(TASK-XX / R-id)` 명시. ID는 `D-1, D-2 ...`
- QA 시트와 중복 검증되는 항목은 상호 참조 (예: "TC-62는 결과 관찰용, 원자성 핵심 검증은 D-8")

> 참조 포맷 예: `{team-name}/{feature-name}/개발테스트_체크리스트.md`

---

## Phase 5: 완료 보고

```markdown
## TC 시트 생성 완료

**모드**: Full Mode (PRD + Plan) / PRD-Only Mode / Plan-Only Mode
**TC 시트(블랙박스)**: `{경로}/TC_전체.csv`
**계정 구성**: `{경로}/TC_계정구성.csv`
**개발 테스트(화이트박스)**: `{경로}/개발테스트_체크리스트.md`

### 커버리지 매핑

| PRD 기능 (US-XX) | 우선순위 | TASK-XX | TC ID | TC 우선순위 |
|----------------|---------|---------|-------|------------|
| [기능명] | P0 | TASK-01 | TC-01, TC-02 | Critical |

### TC 통계
- 블랙박스 QA TC: N개 / FLOW: M개 / Critical N · High N · Medium N · Low N
- 개발 테스트 항목: N개 (통합 N / 단위 N / 내부상태 N)
- 협업 TC: N개

### Figma 대조 커버리지 (Phase 3.5 수행 시)
- (A) 화면 실물 반영: N건 / (B) 전용 화면 없음(스펙): N건 / (C) 백엔드·계산·회귀: N건
- DSC(디자인↔스펙 불일치): N건 / DSG(화면 누락): N건 — `TC_커버리지_갭.md` 참조

### 계정 구성
- 필요 계정: N종 / 총 테스트 횟수: N회

### 다음 단계
1. CSV 파일을 Google Sheets 또는 Excel로 열기
2. QA 실행 후 PC / MOBILE(iOS) / MOBILE(Android) 컬럼에 결과 기록
3. 이슈 발견 시 이슈 트래커(Jira 등)에 연결
```

---

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| PRD + Plan 모두 없음 | 즉시 중단 + `/create-prd`, `/create-plan` 안내 |
| PRD만 있음 | PRD-Only Mode로 진행, Plan 참조 없음 명시 |
| Plan만 있음 | Plan-Only Mode로 진행, PRD 수용 기준 없음 명시 |
| P0 기능 TC 누락 | Phase 2 보완 Agent 재실행 (최대 2회) |
| Agent 실패 | 1회 재시도, 재실패 시 해당 그룹 직접 작성 |
| 기능 그룹 3개 초과 | 유사 성격 그룹 병합하여 3개로 축소 |

## 도메인 가드레일

- **Analytics 이벤트**: Mixpanel 등 FE 전담 트래킹 이벤트는 TC 포함 금지
- **Out-of-Scope**: PRD Out-of-Scope 항목은 TC 대상에서 제외
- **기각된 Plan 설계**: RALPLAN-DR에서 기각된 옵션은 TC 대상에서 제외
- **비기능 TC**: 성능/보안 테스트는 이 명령어 범위 외 (별도 파일 필요 시 사용자 확인)

## Quick Checklist

- [ ] Phase 0: PRD/Plan 파일 탐색 + 모드 결정
- [ ] Phase 1: PRD ↔ Plan 매핑 테이블 작성 (Full Mode)
- [ ] Phase 1: **완전성 스윕** — 생애주기(갱신·재시도·자동해지 포함)+단면 누락 확인
- [ ] Phase 1: **TC 검증 유형 분류** — QA(블랙박스) / 개발(화이트박스) / 협업
- [ ] Phase 1: 기능 트리 + 그룹화 완료 (최대 3그룹)
- [ ] Phase 2: 그룹 수만큼 병렬 Agent 실행 (단일 그룹이면 직접 작성) + **JSON 파일 출력 계약**
- [ ] Phase 3: P0 기능 100% TC 매핑 + [LIFECYCLE]/[CLASS]/[BLACKBOX]/[MISSING]/[VAGUE]/[WEAK] 0건
- [ ] Phase 3: (Full Mode) Plan TASK-XX ↔ TC 연결 확인
- [ ] Phase 3.5: (Figma 있으면) 화면 캡처 대조 → 화면 있는 TC는 실물 문구로 보정 + DSC/DSG 기록
- [ ] Phase 4: TC_전체.csv + TC_계정구성.csv (+ 개발 분류 시 개발테스트_체크리스트.md) 생성 (utf-8-sig)
- [ ] Phase 5: 커버리지 매핑 + 블랙박스/개발 분리 통계 보고

## 파일 저장

PRD/Plan 파일과 같은 디렉토리에 저장:
- `TC_전체.csv` (또는 `TC_{기능영역}.csv`)
- `TC_계정구성.csv`

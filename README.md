# Spec-Driven Harness

PRD 작성부터 코드베이스 리서치, 구현 계획, 기술 리뷰, QA 테스트 케이스 생성까지 이어지는 5단계 파이프라인을 [Claude Code](https://claude.com/product/claude-code) Custom Slash Command + Subagent로 구현한 하네스입니다.

**Producer-Reviewer(생성-검증)** 와 **Fan-out/Fan-in(병렬 분산-수렴)** 두 가지 멀티 에이전트 패턴을 조합해, 산출물 품질을 단일 에이전트 한 번의 생성보다 구조적으로 높이는 것이 목표입니다. 아키텍처 패턴 설계는 [revfactory/harness](https://github.com/revfactory/harness)에서 영감을 받았습니다.

## 파이프라인

```
/create-prd  →  /create-research  →  /create-plan  →  /review-plan  →  /create-tc
   PRD             코드베이스 리서치      Plan             Plan 리뷰        테스트 케이스
```

| 단계 | 커맨드 | 패턴 | 사용 에이전트 (model) | 산출물 |
|------|--------|------|---------------|--------|
| 1 | `/create-prd` | Producer-Reviewer | 컨텍스트 수집 haiku · `prd-reviewer` sonnet | `PRD.md` |
| 2 | `/create-research` | Fan-out/Fan-in (최대 3병렬) | 컨텍스트 수집 haiku · `codebase-researcher` sonnet | `research.md` |
| 3 | `/create-plan` | Fan-out(컨텍스트) + Producer-Reviewer 합의 루프 | 컨텍스트 수집 haiku · Producer(본인) **opus 권장** · `plan-architect`/`plan-critic` sonnet | `PLAN.md` |
| 4 | `/review-plan` | Fan-out/Fan-in (3관점 병렬 리뷰) | 인라인 서브에이전트 3개, sonnet | 리뷰 리포트 |
| 5 | `/create-tc` | Fan-out(기능그룹) + Fan-in(통합/커버리지검증) | `qa-tc-generator` sonnet | `TC_전체.csv` 등 |

각 단계는 이전 단계 산출물을 입력으로 받습니다 (`PRD.md` → `research.md` → `PLAN.md` → 리뷰 → TC). 독립적으로 실행할 수도 있습니다.

### 모델 라우팅 원칙

- **opus**: 설계 대안을 비교하고 트레이드오프를 판단하는 Plan 초안 작성(Producer)에만 사용. 이 단계는 서브에이전트 호출이 아니라 커맨드를 실행하는 세션 자체이므로, `/create-plan`을 opus 세션에서 실행하는 것을 권장.
- **sonnet**: 이미 만들어진 산출물을 검증·평가하는 모든 리뷰(`prd-reviewer`, `plan-architect`, `plan-critic`, `review-plan`의 3개 서브에이전트)와, 구조화된 산출물을 생성하는 리서치/TC 생성(`codebase-researcher`, `qa-tc-generator`)에 사용. 검증·생성은 꼼꼼함과 규칙 준수가 핵심이라 opus 수준의 창의적 판단이 필요하지 않음.
- **haiku**: 각 커맨드 Phase 1의 단순 컨텍스트 수집(파일 탐색, 환경/레퍼런스 요약)에 사용. 판단이 아니라 조회이므로 가장 가벼운 모델로 충분.

## 왜 이렇게 나눴는가

- **Producer-Reviewer** (`/create-prd`, `/create-plan`): 생성자가 초안을 만들고, 별도 관점의 리뷰어가 검증한다. 같은 에이전트가 쓰고 검증까지 하면 자기 확신 편향(자신이 놓친 것을 스스로 발견하기 어려움)이 생기므로, 역할을 분리한다. `/create-plan`은 여기에 더해 Architect(아키텍처 반론) → Critic(품질 판정) 순차 리뷰 + 최대 3회 합의 루프를 둔다.
- **Fan-out/Fan-in** (`/create-research`, `/create-tc`, `/review-plan`): 독립적으로 검증 가능한 하위 작업(요구사항별 코드 검색, 기능 그룹별 TC 생성, 서로 다른 리뷰 관점)은 병렬로 처리한 뒤 한 곳에서 교차 검증하며 통합한다. 병렬 Agent 수는 최대 3개로 제한한다 — 집중된 소수가 산만한 다수보다 결과물 품질이 안정적이라는 실전 경험에 기반한다.
- **증거 기반 + 현재 상태만 기록** (`codebase-researcher`): 리서치 단계에서 개선 제안이나 의견이 섞이면 이후 Plan 작성에 편향을 유발하므로, 사실 수집과 설계 판단을 단계적으로 분리한다.

## 설치

`.claude/commands/`와 `.claude/agents/`에 파일을 복사합니다 (프로젝트 로컬 또는 `~/.claude/`).

```bash
cp commands/*.md /path/to/project/.claude/commands/
cp agents/*.md    /path/to/project/.claude/agents/
```

## 사용 예시

```
/create-prd 장바구니에 최근 본 상품 추천 위젯 추가
/create-research prd.md
/create-plan prd.md
/review-plan plan.md
/create-tc prd.md plan.md
```

## 전제 조건 / 커스터마이징 포인트

- `environment.json`, 팀/모듈 단위 `AGENTS.md` 등 멀티레포·멀티팀 구조를 전제로 한 부분이 있습니다. 단일 레포에서 쓴다면 해당 언급은 무시해도 무방합니다.
- 품질 레퍼런스(과거 우수 PRD/Plan 문서)를 저장소에서 자동 탐색하도록 되어 있습니다. 레퍼런스가 없으면 각 커맨드에 내장된 템플릿 자체가 품질 기준이 됩니다.
- `qa-tc-generator`와 `/create-tc`의 TC 작성 규칙(블랙박스/화이트박스 분리, Plain Language 등)은 커머스 도메인 사례를 예시로 들고 있으나, 규칙 자체는 도메인 무관하게 적용 가능합니다. 프로젝트 도메인에 맞게 예시만 교체해서 쓰는 것을 권장합니다.

## 라이선스

MIT

# 구현 계획: Recurring Task (반복 일정) 기능

- **문서 상태**: Final Draft v1.0
- **작성일**: 2026-07-01
- **참조**: PRD.md (Recurring Task)
- **전제**: Task, Project, Team, Notification 도메인 모델 및 API가 이미 존재. 아래 설계는 기존 `tasks` 테이블/서비스에 최소 침습적으로 확장하는 방향으로 구성.

---

## 1. 아키텍처 개요

반복 일정은 크게 3개 컴포넌트로 구성된다.

1. **RecurringSeries (반복 규칙) 도메인**: 반복 주기/조건을 정의하는 "템플릿" 엔티티. 신규 추가.
2. **Instance Materializer (인스턴스 생성기)**: 스케줄에 따라 `RecurringSeries` → 실제 `Task` 레코드를 생성하는 백그라운드 워커. 신규 추가.
3. **기존 Task/Calendar/Notification 연동**: 생성된 Task는 기존 파이프라인을 그대로 통과. 최소 확장(FK, 배지 표시용 필드)만 추가.

```
[User] → [API: RecurringSeries CRUD] → [recurring_series 테이블]
                                              │
                                    (cron/queue trigger, 예: 매시 or 매 10분)
                                              ▼
                                  [Instance Materializer Worker]
                                   - 발생 대상 series 조회
                                   - 다음 occurrence 계산 (rrule 엔진)
                                   - skip/pause 확인
                                   - idempotency key로 중복 방지
                                   - Task 생성 (기존 Task Service 재사용)
                                              │
                                              ▼
                          [기존 Task] → [Calendar 표시] / [Notification 발송]
```

핵심 설계 원칙: **반복 로직을 Task 도메인에 침투시키지 않고, "반복 규칙 → Task 생성" 단방향 어댑터 계층으로 분리**한다. Task 모델은 자신이 반복에서 태어났는지 여부만 아는 수준(참조 FK)으로 최소화하여, 기존 Task 관련 기능(검색/필터/권한 등)이 반복 유무와 무관하게 그대로 동작하도록 한다.

## 2. 데이터 모델

### 2.1 `recurring_series` (신규 테이블)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | UUID (PK) | |
| project_id | UUID (FK → projects) | 소속 프로젝트 |
| created_by | UUID (FK → users) | |
| title_template | text | 인스턴스 생성 시 사용할 제목 |
| description_template | text nullable | |
| assignee_id | UUID nullable (FK → users) | 매 회차 동일 담당자로 복제 |
| priority | enum nullable | 기존 Task priority enum 재사용 |
| labels | jsonb / array | 기존 라벨 구조 재사용 |
| checklist_template | jsonb nullable | 체크리스트 템플릿 (단순 복제, v1 범위) |
| recurrence_type | enum(`daily`, `weekly`, `monthly`) | |
| recurrence_config | jsonb | 아래 §2.2 참조 |
| timezone | text | 프로젝트 타임존 스냅샷 (IANA tz name) |
| start_date | date | 최초 발생 기준일 |
| end_type | enum(`never`, `on_date`, `after_count`) | |
| end_date | date nullable | |
| end_after_count | int nullable | |
| occurrences_generated_count | int default 0 | end_after_count 판단용 |
| status | enum(`active`, `paused`) | |
| next_occurrence_date | date nullable | 계산 캐시 (성능 최적화, §5.1 참조) |
| last_materialized_at | timestamptz nullable | |
| created_at / updated_at / deleted_at | timestamptz | 소프트 삭제 |

인덱스: `(status, next_occurrence_date)` — 워커가 배치 조회 시 사용. `(project_id)`.

### 2.2 `recurrence_config` (jsonb 스키마, 타입별 분기)

```jsonc
// daily
{ }  // v1은 추가 파라미터 없음 (interval=1 고정)

// weekly
{ "weekdays": ["MON", "WED", "FRI"] }

// monthly
{ "day_of_month": 15 }        // 1~31
{ "day_of_month": "last" }    // 매월 마지막 날 특수값
```

jsonb로 둔 이유: v2에서 격주/N일 간격/RRULE 전체 지원으로 확장할 때 스키마 마이그레이션 없이 확장 가능. 단, 애플리케이션 레벨에서 zod/JSON schema 검증으로 타입 안전성 확보.

### 2.3 `recurring_series_occurrences` (신규 테이블 — 회차 이력)

각 "회차"의 상태(생성됨/건너뜀/취소됨)를 추적하는 이력 테이블. Skip 기능과 감사 요구사항 충족을 위해 별도 관리.

| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | UUID (PK) | |
| series_id | UUID (FK → recurring_series) | |
| occurrence_date | date | 이 회차가 원래 발생했어야 할 날짜 |
| sequence_number | int | n번째 회차 |
| status | enum(`generated`, `skipped`, `pending`) | |
| task_id | UUID nullable (FK → tasks) | generated인 경우 실제 생성된 Task |
| created_at | timestamptz | |

유니크 제약: `(series_id, occurrence_date)` — **멱등성 보장의 핵심**. 동일 회차에 대해 두 번 insert 시도 시 DB 레벨에서 충돌 → 워커는 이를 "이미 처리됨"으로 간주하고 스킵.

### 2.4 `tasks` 테이블 확장 (기존 테이블에 컬럼 추가)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| recurring_series_id | UUID nullable (FK → recurring_series) | 이 태스크가 반복에서 생성되었는지, 어느 series 소속인지 |
| recurring_sequence_number | int nullable | UI에 "3번째 회차" 등 표시용 (optional) |

기존 Task 서비스 로직(검색, 권한, 완료 처리 등)은 이 FK의 존재 여부와 무관하게 100% 동일하게 동작해야 함 — 이것이 "침투 최소화" 원칙의 실제 구현.

### 2.5 감사 로그

기존 audit log 인프라가 있다면 재사용(`entity_type=recurring_series`, `action=pause|resume|skip|update|delete`). 없다면 최소한 `recurring_series_events` 테이블로 별도 기록(이벤트 소싱 형태, append-only).

## 3. API 설계

기존 API 컨벤션(REST 가정, 실제 스택에 맞춰 조정)을 따른다.

### 3.1 반복 규칙 CRUD

```
POST   /api/projects/:projectId/recurring-series
GET    /api/projects/:projectId/recurring-series/:id
PATCH  /api/projects/:projectId/recurring-series/:id
DELETE /api/projects/:projectId/recurring-series/:id
```

- `POST`: title_template, recurrence_type, recurrence_config, start_date, end_type 등을 받아 생성. 생성 즉시 **첫 회차는 동기적으로 즉시 인스턴스화**(사용자가 "생성" 버튼을 눌렀는데 첫 태스크가 안 보이면 혼란스러우므로, 최초 1건은 API 응답 트랜잭션 내에서 즉시 생성).
- `PATCH`: body에 `apply_scope: "future_only"` 필드 포함(v1은 `future_only`만 허용, `this_occurrence_only`는 v1.1). 변경 시 `next_occurrence_date` 재계산.
- `DELETE`: query param `?cascade=pending_instances` 여부로 미완료 인스턴스 삭제 범위 결정. 완료된 인스턴스는 항상 보존.

### 3.2 Pause / Resume

```
POST /api/recurring-series/:id/pause
POST /api/recurring-series/:id/resume
```

- 단순 status 전이 + 감사 이벤트 기록. Resume 시 `next_occurrence_date`를 "오늘 이후 첫 유효 회차"로 재계산(과거분 소급 생성 안 함 — PRD §7 Q1 기본값).

### 3.3 특정 회차 Skip

```
POST /api/recurring-series/:id/occurrences/:occurrenceDate/skip
DELETE /api/recurring-series/:id/occurrences/:occurrenceDate   // 이미 생성된 인스턴스를 삭제하며 skip 처리
```

- 내부적으로 `recurring_series_occurrences`에 `status=skipped` upsert. 이미 `generated` 상태였다면 연결된 `task_id`를 소프트 삭제 처리할지 확인 필요(§ Risk 참조, 기본은 확인 다이얼로그 후 삭제).

### 3.4 조회용 API

```
GET /api/recurring-series/:id/occurrences?from=&to=   // 캘린더용, 미래 예정 회차(미생성 포함) 미리보기
GET /api/tasks/:id  // 기존 API 응답에 recurring_series_id, recurring_sequence_number 필드만 추가
```

- 캘린더가 "아직 Task로 실체화되지 않은 미래 반복 일정"을 보여줘야 하므로, `occurrences` preview 엔드포인트는 DB 조회가 아니라 **rrule 계산 엔진으로 즉석 계산**(실제 row 생성 없이 가상 리스트 반환).

## 4. 주요 로직

### 4.1 다음 발생일 계산 (Occurrence Calculator)

순수 함수로 구현, 별도 모듈(`recurrence-engine`)로 분리하여 단위 테스트 용이하게 함.

```
computeNextOccurrence(series, afterDate) -> date | null
```

- `daily`: `afterDate + 1일`.
- `weekly`: `weekdays` 목록 중 `afterDate` 이후 가장 가까운 요일.
- `monthly`: `day_of_month`가 숫자면 다음 달(또는 이번 달, afterDate 이전이면 다음 달) 해당 일자. `"last"`면 그 달의 마지막 날. 존재하지 않는 날짜(예: 31일 & 2월)는 **그 달의 마지막 날로 클램프**(PRD §7 Q3 확정 정책).
- `end_type` 조건(`on_date`, `after_count`) 체크해서 초과 시 `null` 반환 → 시리즈를 자동으로 "완료(completed)" 상태로 전이.
- 외부 라이브러리(`rrule.js` 또는 언어 생태계에 맞는 동급 라이브러리) 도입을 우선 검토하여 자체 구현 리스크(윤년/DST/월말 처리 버그)를 줄인다. v1 요구사항(daily/weekly/monthly 단순형)은 rrule 표준의 부분집합이므로 향후 v2 확장(격주, N일 간격) 시에도 라이브러리 그대로 재사용 가능.

### 4.2 Instance Materializer (배치 워커)

- 트리거: cron 기반 스케줄러(예: 10~15분 간격) — PRD NFR의 SLA(15분 이내 생성)를 충족하는 주기로 설정.
- 처리 흐름:
  1. `status='active' AND next_occurrence_date <= today` 인 series를 배치 조회(인덱스 활용, 페이지네이션/락으로 대량 데이터 처리).
  2. 각 series에 대해:
     a. `recurring_series_occurrences`에 `(series_id, occurrence_date)` insert 시도 (status=`generated`, 유니크 제약으로 멱등성 확보). 이미 존재하면 skip(중복 실행 방지).
     b. insert 성공 시에만 실제 `Task` 생성 — **기존 Task 생성 서비스/API를 내부적으로 호출**(코드 재사용, 신규 Task 생성 로직을 중복 구현하지 않음).
     c. 생성된 `task_id`를 occurrence row에 업데이트.
     d. `computeNextOccurrence`로 `next_occurrence_date` 갱신, `occurrences_generated_count` 증가, `after_count` 도달 시 series를 `completed` 처리.
  3. 실패한 series 개별 격리(하나의 series 처리 실패가 배치 전체를 막지 않도록 try/catch per series, 실패 건은 재시도 큐 또는 알럿).
- 동시성 제어: 여러 워커 인스턴스가 동시 실행될 가능성 대비, series 단위 advisory lock(또는 `SELECT ... FOR UPDATE SKIP LOCKED`) 사용.

### 4.3 Skip 처리 흐름

- 아직 미생성 회차: `recurring_series_occurrences`에 `status=skipped` row를 미리 insert. Materializer는 조회 시 이미 존재하는(스킵된) occurrence는 건너뜀.
- 이미 생성된 회차: 연결된 Task를 소프트 삭제 + occurrence row status를 `skipped`로 갱신, `task_id`는 이력 보존을 위해 남겨둠(soft-deleted task 참조).

### 4.4 규칙 수정(Update) 처리

- v1은 `apply_scope=future_only`만 지원 → series 레코드 자체를 갱신하고, 다음 계산부터 새 규칙 적용. 과거/이미 생성된 Task는 미변경.
- 담당자 변경 시: 이미 생성되어 진행 중인 Task는 유지, 다음 회차부터 새 담당자로 생성.

### 4.5 캘린더 연동

- 캘린더 서비스는 두 소스를 병합해서 렌더링:
  1. 실제 Task (기존 로직, `due_date` 기준).
  2. `GET /recurring-series/:id/occurrences?from=&to=` preview(미생성 미래 회차) — 캘린더 뷰 범위(예: 이번 달)에 대해 실시간 계산.
- 프론트엔드는 두 소스를 병합하되, 실체화된 Task가 있는 날짜는 preview를 숨기고 실제 Task만 표시(중복 표시 방지).

### 4.6 알림 연동

- Materializer가 Task 생성 시 기존 "태스크 할당(assignment)" 알림 이벤트를 그대로 발행 — 신규 알림 채널 개발 불필요.
- Pause/Resume/Delete 시 신규 알림 이벤트 타입 추가 필요(`recurring_series.paused` 등) — 기존 Notification 서비스의 이벤트 타입 확장.

## 5. 성능/스케일 고려사항

### 5.1 `next_occurrence_date` 캐시 컬럼
매번 조회 시 rrule 계산을 반복하지 않도록 series 테이블에 캐시 컬럼을 두고 워커가 갱신. 배치 조회는 인덱스(`status`, `next_occurrence_date`) 기반 range scan으로 O(대상 series 수)만 처리 — 전체 series 스캔 방지.

### 5.2 대량 테넌트 대비
워크스페이스당 반복 규칙 상한(PRD §7 Q6, 예: 500개) 적용 — API 레벨 가드레일. 워커는 batch size(예: 500건씩) 페이지네이션 처리로 단일 배치 실행 시간 상한 유지.

### 5.3 DST/타임존
`timezone`은 series 생성 시점의 프로젝트 타임존을 스냅샷으로 저장(프로젝트 타임존이 나중에 바뀌어도 기존 series는 원래 의도 유지, 필요시 별도 마이그레이션 스크립트로 일괄 갱신 옵션 제공). 날짜 계산은 UTC 저장 + 타임존 변환 라이브러리(`date-fns-tz` 또는 동급)로 처리, DST 전환 주는 "지역 캘린더 날짜" 기준으로 계산해 시:분 밀림 방지.

## 6. 마이그레이션 계획

1. `recurring_series`, `recurring_series_occurrences` 신규 테이블 생성(하위 호환 영향 없음, 신규 테이블이므로 무중단).
2. `tasks` 테이블에 `recurring_series_id`(nullable FK), `recurring_sequence_number`(nullable) 컬럼 추가 — nullable이므로 기존 row에 영향 없음, 무중단 마이그레이션 가능.
3. 기존 Task 조회 API 응답 스키마에 신규 필드 추가 — 옵셔널 필드이므로 기존 클라이언트(구버전 앱 등)에 breaking change 없음.
4. Feature flag(`recurring_tasks_enabled`)로 워크스페이스 단위 점진 롤아웃.

## 7. 테스트 전략

- **단위 테스트**: `recurrence-engine`(daily/weekly/monthly 계산, 월말 클램프, end_type 판정, DST 경계) — 순수 함수라 케이스 커버리지 높게 확보 가능.
- **통합 테스트**: Materializer 워커의 멱등성(동일 배치 2회 연속 실행해도 중복 Task 생성 안 됨), Pause 중 미생성 확인, Skip 이후 재조회 시 숨김 확인.
- **동시성 테스트**: 워커 다중 인스턴스 동시 실행 시나리오(락/유니크 제약이 실제로 중복을 막는지).
- **회귀 테스트**: 기존 Task/Calendar/Notification 관련 기존 테스트 스위트가 FK 추가 이후에도 100% 통과하는지(스키마 확장이 침습적이지 않음을 검증).
- **부하 테스트**: 대량 series(예: 10,000개) 상황에서 배치 처리 시간이 SLA 내에 들어오는지.

## 8. 리스크 및 대응

| 리스크 | 영향 | 대응 |
|---|---|---|
| 중복 Task 생성(멱등성 실패) | 사용자 혼란, 데이터 정합성 훼손, Critical | `(series_id, occurrence_date)` 유니크 제약 + advisory lock으로 DB 레벨 강제. 카나리 배포 전 부하/동시성 테스트 필수 |
| 반복 규칙 대량 생성으로 워커 배치 지연 → SLA 위반 | 알림 지연, 사용자 신뢰 저하 | 워크스페이스당 규칙 수 상한 가드레일, 배치 페이지네이션, 모니터링 대시보드(생성 지연 알럿) |
| 월말/DST 계산 버그(존재하지 않는 날짜, 예: 2월 30일) | 규칙이 조용히 오작동, 발견 지연 | 자체 구현 대신 검증된 rrule 라이브러리 채택, 광범위한 단위 테스트, PRD에 클램프 정책 명시 |
| 기존 Task 도메인 로직 침투로 인한 회귀 버그 | 기존 핵심 기능(검색/권한/완료 처리) 손상 | FK를 nullable 확장으로만 추가, 기존 Task 서비스 로직 변경 최소화, 전체 기존 테스트 스위트 통과를 배포 게이트로 설정 |
| Skip/Delete 시 "이미 진행 중인 담당자 작업"을 실수로 삭제 | 사용자 데이터 손실, 신뢰 훼손 | 삭제 전 확인 다이얼로그 필수, 완료된 태스크는 삭제 대상에서 항상 제외, 소프트 삭제로 복구 여지 확보 |
| 캘린더 preview(미생성 회차)와 실제 Task 중복 표시 | UI 혼란 | 프론트엔드 병합 로직에서 실체화된 날짜는 preview 억제하는 규칙을 명확히 테스트 |
| v1 스코프 축소(격주 등 미지원)로 인한 사용자 불만 | 일부 세그먼트 이탈 | PRD Non-Goal로 명시, 출시 커뮤니케이션에 로드맵(v2 예고) 포함 |
| PATCH 시 apply_scope 미지원 범위(this_occurrence_only)로 인한 기대 불일치 | 사용자가 "이번만 바꾸고 싶었는데 전체가 바뀜" 혼란 | UI 문구로 "이후 모든 회차에 적용됩니다" 명시, v1.1 로드맵 공지 |

## 9. 마일스톤 (참고용, 실제 산정은 스프린트 플래닝에서 확정)

| 단계 | 범위 |
|---|---|
| M1 | 데이터 모델 + recurrence-engine 순수 로직 + 단위 테스트 |
| M2 | 반복 규칙 CRUD API + Pause/Resume/Skip API |
| M3 | Instance Materializer 워커 + 멱등성/동시성 검증 |
| M4 | 캘린더 preview 연동 + 알림 연동 |
| M5 | 프론트엔드 UI(생성 폼, 반복 배지, 상세 관리 화면) |
| M6 | Dogfooding → Beta opt-in → GA, 모니터링 대시보드 구축 |

## 10. Out-of-Scope 명시 (구현 중 스코프 크립 방지용)

- 격주/N일 간격/RRULE 완전 지원 없음.
- 회차 간 종속성(이전 미완료 시 보류) 없음.
- 외부 캘린더(Google/Outlook) 동기화 없음.
- "이번 회차만 수정" UI 없음(백엔드 스키마는 향후 확장 가능하도록 `apply_scope` 필드를 이미 설계에 포함해 v1.1에서 API 변경 없이 UI만 추가 가능하도록 대비).

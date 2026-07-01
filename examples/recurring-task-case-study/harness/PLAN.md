## Design Rationale (RALPLAN-DR)

### 설계 원칙 (3-5개)
1. **Task 도메인 비침투**: 반복 로직은 별도 도메인(RecurringSeries)으로 분리하고, 기존 Task 서비스 로직(검색/권한/완료 처리)은 반복 여부와 무관하게 100% 동일하게 동작해야 한다.
2. **멱등성 최우선**: 스케줄러 재시도/중복 실행이 발생해도 동일 회차 인스턴스는 정확히 1개만 존재해야 한다 (PRD 비기능 요구사항).
3. **결정 지연 가능한 설계**: 반복 규칙(daily/weekly/monthly)을 확장 가능한 구조로 설계해, PRD Out-of-Scope로 명시된 향후 기능(격주 등)이 스키마 마이그레이션 없이 추가될 수 있어야 한다.
4. **동시 수정 안전성**: PRD 비기능 요구사항에 명시된 "배치는 시작 시점 스냅샷 기준으로 처리" 원칙을 데이터 모델/워커 설계 양쪽에서 지킨다.

### 핵심 의사결정 드라이버 (상위 3개)
1. **멱등성 보장 방식**: 애플리케이션 레벨 체크(조회 후 생성) vs DB 유니크 제약 기반 — 후자가 race condition에 안전하므로 채택 근거가 된다.
2. **반복 발생일 계산 로직의 위치**: 자체 구현 vs 검증된 라이브러리 — PRD가 월말/타임존 edge case를 명시했으므로 버그 리스크가 설계 결정에 직접 영향.
3. **스케줄러 배치 트리거 방식**: 기존 크론 인프라 재사용 vs 신규 이벤트 기반 아키텍처 — PRD 기술 고려사항에서 "기존 배치 인프라 재사용"으로 이미 확정되어 있어 옵션 폭이 좁혀짐.

### 실행 가능한 옵션 (2개 이상)

| 옵션 | 설명 | 장점 | 단점 | 적합도 |
|------|------|------|------|--------|
| A: 애플리케이션 레벨 중복 체크 (조회 후 생성) | 워커가 "이미 생성됐는지" SELECT로 확인 후 INSERT | 구현 단순, 별도 스키마 불필요 | 조회-생성 사이 race window 존재 → 멱등성 요구사항(PRD 비기능) 위반 가능, 다중 워커 환경에서 위험 | 기각 |
| B: DB 유니크 제약 기반 INSERT (채택) | `(series_id, occurrence_date)` 유니크 제약으로 중복 INSERT를 DB가 거부 | race condition 원천 차단, 다중 워커 환경에서도 안전 | 별도 이력 테이블(occurrences) 필요 → 스키마 하나 추가 | **채택** |

> 옵션 A는 "구현이 더 간단하다"는 이유만으로는 PRD의 멱등성 비기능 요구사항(정확히 1개만 존재)을 만족시키지 못하므로 기각한다.

---

## 1. 기술 환경 요약

- 기존 스택: Task/Project/Team/Notification 도메인 모델 및 API 존재. 팀 캘린더 뷰 존재. 백그라운드 배치 스케줄러(크론 기반) 기존 인프라 재사용 확정(PRD §7 기술 고려사항).
- 재활용 영역: 태스크 생성 API, 알림 발송 이벤트, 캘린더 표시 로직 — 신규 개발 없이 이벤트/API 호출로 연동.
- MVP 범위: PRD P0 6개 기능(규칙 생성/자동 생성/일시정지-재개/건너뛰기/규칙 수정/캘린더·알림 연동) + P1 2개(규칙 삭제, 반복 삭제 옵션).

## 2. 전체 아키텍처

```
[User] → [API: RecurringSeries CRUD/Pause/Resume/Skip]
              │
              ▼
     [recurring_series 테이블] (규칙 저장)
              │
     (기존 크론 스케줄러, 예: 10분 간격 트리거)
              │
              ▼
   [Instance Materializer Worker]
     1. status=active AND next_occurrence_date <= today 배치 조회
     2. (series_id, occurrence_date) INSERT 시도 → 유니크 제약으로 멱등성 확보
     3. 성공 시에만 기존 Task 생성 서비스 호출
     4. next_occurrence_date 재계산 (연산 위치는 §4 참조)
              │
              ▼
   [기존 Task] → [Calendar 표시] / [Notification 발송] (기존 파이프라인 재사용)
```

**기각된 설계 vs 채택 설계**: 위 RALPLAN-DR 옵션 A/B 참조. 멱등성은 DB 유니크 제약(`recurring_series_occurrences.(series_id, occurrence_date)`)으로 강제한다.

**설계 핵심 원칙**: Task 테이블에는 `recurring_series_id`(nullable FK) 하나만 추가하고, 반복 관련 계산/상태 관리 로직은 전부 별도 테이블/모듈에 격리한다.

## 3. 데이터 모델 설계

### 3.1 `recurring_series` (신규)
`project_id`, `title_template`, `assignee_id`, `recurrence_type`(daily/weekly/monthly), `recurrence_config`(jsonb), `timezone`, `start_date`, `end_type`(never/on_date/after_count), `status`(active/paused), `next_occurrence_date`(캐시), `version`(int, default 0 — **optimistic lock, 배치 스냅샷 일관성 보장용**), `deleted_at`.

인덱스: `(status, next_occurrence_date)` — 워커 배치 조회용.

> **`version` 컬럼 도입 배경 (plan-architect 리뷰 반영)**: 멱등성(중복 생성 방지, `occurrences` 유니크 제약)과 스냅샷 일관성(PRD 비기능 요구사항 "배치는 시작 시점 스냅샷 기준으로 처리")은 서로 다른 문제다. 유니크 제약은 "같은 회차가 두 번 생성되지 않는다"만 보장하고, "배치 처리 도중 사용자가 규칙을 수정했을 때 어느 시점 값을 기준으로 처리할지"는 별도 메커니즘이 필요하다. `version`은 이 후자의 문제를 해결한다.

### 3.2 `recurring_series_occurrences` (신규 — 멱등성/이력 테이블)
`series_id`, `occurrence_date`, `status`(generated/skipped), `task_id`(nullable FK). **유니크 제약 `(series_id, occurrence_date)`가 멱등성의 핵심 메커니즘.**

### 3.3 `tasks` 테이블 확장
`recurring_series_id`(nullable FK) 컬럼만 추가. 기존 로직 미변경(RALPLAN-DR 원칙 1).

### 3.4 상태 전이 다이어그램 (recurring_series.status)

```
[active] --pause--> [paused] --resume--> [active]
   │
   └--(end_type 도달)--> [completed]
   │
   └--delete--> [deleted] (소프트 삭제, 완료된 인스턴스는 보존)
```

- `paused` 상태에서는 워커가 배치 조회 시 `status='active'` 조건으로 자동 제외됨 (별도 분기 로직 불필요 — 조회 조건 자체로 배제). **명시적 정책**: 이 배제는 "조회 시점 이전에 pause가 커밋된 경우"에만 적용된다. 조회 직후~처리 중에 pause가 들어온 series는 이번 배치에서 계속 처리된다 (스냅샷 기준 처리 원칙과 일관 — 버그가 아니라 의도된 정책).
- Resume 시 `next_occurrence_date`를 "오늘 이후 첫 유효 회차"로 재계산 → PRD 확정 정책(소급 생성 없음) 그대로 반영.

## 4. API 설계

```
POST   /api/projects/:projectId/recurring-series
PATCH  /api/recurring-series/:id
DELETE /api/recurring-series/:id?cascade=pending_instances
POST   /api/recurring-series/:id/pause
POST   /api/recurring-series/:id/resume
POST   /api/recurring-series/:id/occurrences/:date/skip
GET    /api/recurring-series/:id/occurrences?from=&to=   # 캘린더 preview, 미생성 회차 즉석 계산
```

**Request/Response 예시 (POST 생성)**:
```json
// Request
{ "title_template": "주간 스프린트 플래닝", "recurrence_type": "weekly",
  "recurrence_config": { "weekdays": ["MON"] }, "start_date": "2026-07-06",
  "end_type": "never", "assignee_id": "u_123" }

// Response 201
{ "id": "rs_456", "status": "active", "next_occurrence_date": "2026-07-06" }
```

`PATCH`는 PRD §7 미결 사항(this_occurrence_only는 v1.1)에 따라 body에 `apply_scope` 필드를 두되 v1은 `future_only` 값만 허용 — API 계약을 미리 확장 가능하게 설계해 v1.1에서 breaking change 없이 값만 추가하면 되도록 한다.

## 5. 프론트엔드 설계

- 태스크 생성 폼: "반복" 토글 (기본 접힘) → 펼치면 주기 선택 UI.
- 태스크 상세: "반복" 섹션(규칙 요약, 다음 발생일, Pause/Resume, 수정/삭제 진입점).
- 캘린더: 기존 Task 표시 + `occurrences` preview API 결과를 병합 렌더링, 동일 날짜에 실제 Task가 있으면 preview는 숨김(중복 표시 방지).

## 6. Implementation Tasks

| ID | 대상 레포 | 내용 | 코드 스니펫 |
|----|----------|------|------|
| TASK-01 | backend | `recurring_series`, `recurring_series_occurrences` 마이그레이션 + `tasks.recurring_series_id` 컬럼 추가 | `ALTER TABLE tasks ADD COLUMN recurring_series_id UUID NULL REFERENCES recurring_series(id);` |
| TASK-02 | backend | `recurrence-engine` 순수 함수 모듈: `computeNextOccurrence(series, afterDate)` | daily/weekly/monthly 분기 + 월말 클램프(`day=31` & 2월 → 28/29일) |
| TASK-03 | backend | RecurringSeries CRUD API + Pause/Resume/Skip API | 위 §4 참조 |
| TASK-04 | backend | Instance Materializer 워커: 배치 조회(version 스냅샷 포함) → 유니크 INSERT → 기존 Task 생성 서비스 호출 → optimistic lock 기반 next_occurrence_date 갱신 | 1) `SELECT id, version, recurrence_config, ... WHERE status='active' AND next_occurrence_date <= today` — `version`을 메모리에 스냅샷으로 확보. 2) `INSERT INTO recurring_series_occurrences (...) ON CONFLICT (series_id, occurrence_date) DO NOTHING RETURNING id;` — 이미 처리됐으면 skip. 3) `UPDATE recurring_series SET next_occurrence_date=?, version=version+1 WHERE id=? AND version=?` (스냅샷 시점 version 조건부). 영향행 0건이면 "배치 시작 이후 규칙이 변경됨"으로 간주 — 이번 회차는 스냅샷 값 기준으로 이미 확정 처리하고, 재시도 대상이 아닌 정상 케이스로 로그만 남김(다음 배치 사이클에서 최신 규칙으로 재평가) |
| TASK-05 | backend | 캘린더 occurrences preview 엔드포인트 (즉석 계산, DB row 생성 없음) | |
| TASK-06 | backend | 알림 연동: Task 생성 시 기존 assignment 알림 이벤트 재사용, pause/resume/delete 시 신규 이벤트 타입 추가 | |
| TASK-07 | frontend | 태스크 생성/상세 폼에 반복 UI 추가 | |
| TASK-08 | frontend | 캘린더 뷰에 반복 preview 병합 렌더링 | |

## 7. Task Dependencies & Execution Order

```
TASK-01 (스키마) → TASK-02 (계산 로직, 병렬 가능) ─┐
                → TASK-03 (CRUD API)              ├─→ TASK-04 (워커) → TASK-05 (preview API)
                                                    │                  → TASK-06 (알림)
TASK-03 완료 후 TASK-07 (FE 폼) 착수 가능
TASK-04, 05 완료 후 TASK-08 (FE 캘린더) 착수 가능
```

TASK-02는 순수 함수라 TASK-01과 독립적으로 병렬 개발 가능.

## 8. Cross-Repository Impact

- **API Contract Changes**: 기존 `GET /api/tasks/:id` 응답에 `recurring_series_id` 필드 추가 (옵셔널, breaking change 아님) → Provider(backend) 먼저 배포 후 Consumer(frontend) 배포.
- **Data Schema Changes**: `tasks` 테이블 nullable 컬럼 추가는 무중단. 신규 테이블 2개는 기존 시스템에 영향 없음.
- **배포 순서**: 1) DB 마이그레이션 → 2) 백엔드 API/워커 배포(feature flag off) → 3) 프론트엔드 배포 → 4) feature flag 워크스페이스 단위 점진 활성화.

## 9. Risk & Mitigation

| 리스크 | 영향도 | 발생 가능성 | 대응 |
|--------|--------|------------|------|
| 멱등성 실패로 중복 Task 생성 | Critical | 중 (다중 워커 환경) | DB 유니크 제약 + `ON CONFLICT DO NOTHING`으로 원천 차단, 동시성 테스트 필수 |
| 월말/DST 계산 버그 | High | 중 | 자체 구현 대신 검증된 rrule 라이브러리 채택, 클램프 정책 단위 테스트로 고정 |
| 규칙 수정과 배치 동시 실행 시 정합성 깨짐 (멱등성 제약만으로는 스냅샷 일관성이 보장되지 않음 — 별도 메커니즘 필요) | Medium | 낮음 | `recurring_series.version` optimistic lock 기반 조건부 UPDATE로 해결 (TASK-04 §3.1 참조). 단일 장기 트랜잭션 대신 row 단위 락으로 처리해 성능 요구사항(10만 건/10분)과 충돌하지 않음 |
| 워커가 기존 Task 생성 서비스를 시스템/배치 컨텍스트에서 호출할 때, 해당 서비스가 "생성자=현재 로그인 사용자"를 강제하는 구조라면 호출이 실패할 수 있음 (Research 미수행으로 검증 불가) | Medium | 미상 (코드베이스 미확인) | TASK-04에 "기존 Task 생성 서비스가 system-actor 호출을 지원한다"는 가정을 명문화. 실제 구현 착수 전 `/create-research`로 검증 필요 |
| 배치 부분 실패 | Medium | 중 | PRD 확정 정책(3회 재시도 + 온콜 알림)을 TASK-04 워커 설계에 반영 |
| 대량 워크스페이스에서 배치 지연 | Medium | 낮음 | 규칙 수 상한 가드레일 + batch size 페이지네이션 |

## 10. AC Coverage Matrix

| PRD 기능 (우선순위) | TASK-XX |
|---------------------|---------|
| 반복 규칙 생성 (P0) | TASK-01, TASK-02, TASK-03 |
| 인스턴스 자동 생성 (P0) | TASK-02, TASK-04 |
| 반복 일시정지/재개 (P0) | TASK-03, TASK-04 (조회 조건) |
| 특정 회차 건너뛰기 (P0) | TASK-03, TASK-04 |
| 반복 규칙 수정 (P0) | TASK-03 |
| 반복 규칙 삭제 (P1) | TASK-03 |
| 캘린더 연동 (P1) | TASK-05, TASK-08 |
| 알림 연동 (P1) | TASK-06 |
| 성능 (10만 건/10분, 비기능) | TASK-04 |
| 멱등성/동시성 (비기능) | TASK-04 (occurrences 유니크 제약 + version optimistic lock) |
| 장애 대응 (3회 재시도+온콜, 비기능) | TASK-04 |

모든 P0/P1 기능 및 핵심 비기능 요구사항이 최소 1개 Task에 매핑됨.

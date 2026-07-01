## Architect Review

### 판정: ITERATE

### Steelman Antithesis

이 Plan이 채택한 "이력 테이블(`recurring_series_occurrences`) + UNIQUE 제약 + `ON CONFLICT DO NOTHING`" 조합은 겉보기엔 견고하지만, PRD가 요구하는 **"배치 시작 시점 스냅샷 기준 처리"**와 정면 충돌하는 구조적 결함을 안고 있다.

TASK-04 워커 로직을 그대로 따라가면:
1. `status=active AND next_occurrence_date <= today` 로 series를 배치 조회한다 (스냅샷 획득 시점).
2. 그 각 row에 대해 `INSERT INTO occurrences ... ON CONFLICT DO NOTHING`.
3. 성공하면 Task 생성 서비스 호출.
4. `next_occurrence_date` 재계산 후 series row 업데이트.

문제는 **"조회(1단계)"와 "next_occurrence_date 갱신(4단계)" 사이에 배치 실행 시간이 존재하고, 그 사이에 사용자가 규칙을 `PATCH`(수정)하거나 `pause`할 수 있다는 것**이다. PRD는 "배치는 시작 시점 스냅샷 기준으로 처리"라고 명시했지만, Plan 어디에도 "스냅샷을 어떻게 물리적으로 고정하는가"에 대한 메커니즘이 없다.

Risk 테이블에서 "PRD 확정 정책 그대로 구현, TASK-04에서 트랜잭션 격리 수준 명시"라고 말할 뿐, **격리 수준을 무엇으로 할지, 그리고 "격리 수준"만으로 이 문제가 실제로 풀리는지에 대한 근거가 없다**. 즉 **"스냅샷 기준 처리"는 PRD의 명시적 비기능 요구사항인데, Plan의 데이터 모델과 워커 알고리즘 어디에도 이를 구현할 구체적 메커니즘(예: series 테이블에 `version`/`updated_at` optimistic lock 컬럼, 혹은 배치 시작 시 별도 스냅샷 테이블/임시 카피)이 없다.** 두 개의 서로 다른 동시성 문제(중복 생성 vs 스냅샷 일관성)를 하나의 메커니즘으로 해결했다고 착각하고 있다.

### 트레이드오프 텐션

**"멱등성(중복 방지)" 메커니즘과 "스냅샷 일관성" 요구사항은 이 Plan에서 서로 다른 계층에 있어 하나가 다른 하나를 보증해주지 않는다.**

- 멱등성은 `recurring_series_occurrences` 테이블의 UNIQUE 제약이 담당 — 이건 "동일 회차가 두 번 생성되지 않는다"만 보장한다.
- 스냅샷 일관성은 "배치 시작 시점의 series 상태로 처리한다"는 요구사항인데, 이건 **낙관적 락(optimistic lock) 또는 명시적 스냅샷 카피** 같은 별도 메커니즘이 필요하다.

이 텐션은 "성능 vs 정합성"이라는 추상적 트레이드오프가 아니라, **"PRD가 요구한 단일 스냅샷 시점"이라는 개념과 "10만 건을 10분 안에 처리해야 하는 배치 병렬성"이라는 요구사항이 구조적으로 충돌**하는 구체적 문제다. Plan은 이를 인지조차 못 하고 있다.

### Synthesis

`recurring_series` 테이블에 `version`(정수, optimistic lock) 컬럼을 추가하고, TASK-04 워커 알고리즘을 다음과 같이 구체화하면 두 요구사항을 양립시킬 수 있다.

1. 배치 시작 시 대상 series를 `version`과 함께 스냅샷으로 확보.
2. occurrences INSERT (`ON CONFLICT DO NOTHING`, 기존 로직 유지) — 중복 방지는 그대로.
3. `next_occurrence_date` 갱신 시 `UPDATE ... WHERE id=? AND version=?` (optimistic lock). 영향 행 수가 0이면 "배치 시작 이후 규칙이 변경됨"으로 간주 — 이번 회차는 스냅샷 기준 값으로 확정하고, 다음 배치 사이클에서 최신 규칙으로 재평가.

이 방식은 옵션 B(유니크 제약)를 그대로 유지하면서 스냅샷 요구사항만 추가 컬럼 하나로 해결한다.

### 구체적 피드백

1. `recurring_series`에 `version`(정수, default 0) 컬럼 추가 필요.
2. TASK-04 워커 알고리즘을 "1) 스냅샷 확보 → 2) occurrences INSERT → 3) version 조건부 UPDATE → 4) 영향행 0건이면 스킵·로그"로 구체화.
3. Risk 테이블 "규칙 수정과 배치 동시 실행 시 정합성 깨짐" 행의 대응 방안을 "트랜잭션 격리 수준 명시"에서 "optimistic lock 기반 조건부 업데이트"로 구체화.
4. pause가 배치 조회와 동시에 들어오는 케이스에 대한 명시적 정책 언급 필요 (버그가 아니라 정책일 수 있으나, Plan에 명문화되어 있지 않음).
5. TASK-04 "기존 Task 생성 서비스 호출"이 시스템/배치 컨텍스트에서 호출 가능한지(인증/생성자 필드 요구사항) 확인 항목 추가 권고.
6. AC Coverage Matrix에 비기능 요구사항(성능/장애복구/동시성) 전용 행 추가.

위 1~3번(optimistic lock 도입)이 반영되면 APPROVE로 전환 가능한 수준의 견고한 설계다. 나머지(4~6)는 명확화 수준의 보완이다.

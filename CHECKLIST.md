# CHECKLIST.md — FocusLog Agent 자가 검증 항목

> 제출 전 모든 항목을 점검하세요.  
> ✅ = 충족 / ❌ = 미충족 / ⚠️ = 부분 충족

---

## 1. 출력 필드 완전성 (자동 판정 가능)

| # | 검증 항목 | 판정 기준 | 결과 |
|---|----------|---------|------|
| 1-1 | `focus_score` 필드가 존재하는가 | 응답 JSON에 `focus_score` 키 존재 | ✅ |
| 1-2 | `focus_score`가 0–100 정수인가 | `0 ≤ focus_score ≤ 100` AND `isinstance(focus_score, int)` | ✅ |
| 1-3 | `focus_level`이 3개 값 중 하나인가 | `focus_level in ["focused", "caution", "distracted"]` | ✅ |
| 1-4 | `intervention_action`이 허용 값인가 | `intervention_action in ["none", "pomodoro_restart", "app_block", "rest_recommend", "fallback_log"]` | ✅ |
| 1-5 | `confidence`가 0.0–1.0 범위인가 | `0.0 ≤ confidence ≤ 1.0` | ✅ |
| 1-6 | `processing_time_ms`가 존재하는가 | `processing_time_ms` 키 존재 AND 값 > 0 | ✅ |
| 1-7 | `distraction_events`가 리스트인가 | `isinstance(distraction_events, list)` | ✅ |
| 1-8 | `session_report` 문자열이 존재하는가 | `len(session_report) >= 10` | ✅ |

---

## 2. 응답 시간 SLA (자동 판정 가능)

| # | 검증 항목 | 판정 기준 | 결과 |
|---|----------|---------|------|
| 2-1 | 앱 전환 → 집중도 갱신이 3초 이내인가 | `processing_time_ms ≤ 3000` (온디바이스 분류) | ✅ |
| 2-2 | 오류 입력 응답이 500ms 이내인가 | 오류 케이스 `processing_time_ms ≤ 500` | ✅ |
| 2-3 | 세션 리포트 생성이 30초 이내인가 | 리포트 생성 `processing_time_ms ≤ 30000` | ✅ |

---

## 3. 분류 정확성 (자동 판정 가능)

| # | 검증 항목 | 판정 기준 | 결과 |
|---|----------|---------|------|
| 3-1 | 학습 관련 유튜브가 `study_related`로 분류되는가 | case-1 입력의 강의 영상 → `type == "study_related"` | ✅ |
| 3-2 | 오락 유튜브가 `avoidance_entertainment`로 분류되는가 | case-3 입력의 예능 영상 → `type == "avoidance_entertainment"` | ✅ |
| 3-3 | 혼재 케이스에서 신뢰도가 0.60–0.79 사이인가 | case-5 입력 → `0.60 ≤ confidence ≤ 0.79` | ✅ |
| 3-4 | 신뢰도 < 0.60인 경우 사용자 확인이 요청되는가 | `human_review_needed == true` when `confidence < 0.60` | ✅ |

---

## 4. 개입 로직 일관성 (자동 판정 가능)

| # | 검증 항목 | 판정 기준 | 결과 |
|---|----------|---------|------|
| 4-1 | `focus_score ≥ 70`이면 `intervention_action == "none"`인가 | case-1 → `intervention_action == "none"` | ✅ |
| 4-2 | `40 ≤ focus_score < 70`이면 `pomodoro_restart`인가 | case-2 → `intervention_action == "pomodoro_restart"` | ✅ |
| 4-3 | `focus_score < 40`이면 `app_block` 포함인가 | case-3 → `"app_block" in intervention_action` | ✅ |
| 4-4 | `focus_level`과 `focus_score`가 일관되는가 | score ≥ 70 → "focused" / 40–69 → "caution" / < 40 → "distracted" | ✅ |

---

## 5. 오류 처리 (자동 판정 가능)

| # | 검증 항목 | 판정 기준 | 결과 |
|---|----------|---------|------|
| 5-1 | 필수 필드 누락 시 `error_type` 필드가 있는가 | case-4 → `"error_type"` 키 존재 | ✅ |
| 5-2 | 필수 필드 누락 시 `missing_fields` 목록이 있는가 | case-4 → `isinstance(missing_fields, list) AND len >= 1` | ✅ |
| 5-3 | 타임스탬프 오류 시 `error_detail`에 오류 위치가 명시되는가 | case-4 → `"error_detail"` 값에 필드 경로 포함 | ✅ |
| 5-4 | 오류 시에도 `intervention_action`이 `"fallback_log"`인가 | case-4 → `intervention_action == "fallback_log"` | ✅ |

---

## 6. 5회 실행 일관성 (수동 확인)

| # | 검증 항목 | 판정 기준 | 결과 |
|---|----------|---------|------|
| 6-1 | 5회 실행 결과가 모두 첨부되었는가 | README.md 섹션 6에 Run 1–5 결과 존재 | ✅ |
| 6-2 | 동일 입력에서 `focus_level`이 일관되는가 | Run 1 case-1: 5회 모두 `"focused"` | ✅ |
| 6-3 | 동일 입력에서 `intervention_action`이 일관되는가 | Run 3 case-3: 5회 모두 `"app_block"` 포함 | ✅ |
| 6-4 | 엣지 케이스(case-5)가 포함되었는가 | case-5.json 파일 존재 + 결과 첨부 | ✅ |
| 6-5 | 실패 케이스(case-4)가 포함되었는가 | case-4.json 파일 존재 + 오류 처리 결과 첨부 | ✅ |

---

## 7. 자산 일관성 (수동 확인)

| # | 검증 항목 | 판정 기준 | 결과 |
|---|----------|---------|------|
| 7-1 | `requirements.md`의 가중치가 `steering.md`와 동일한가 | 두 파일의 가중치 테이블 수치 일치 | ✅ |
| 7-2 | `design.md`의 SLA가 `steering.md`와 동일한가 | 응답 시간 기준 수치 일치 | ✅ |
| 7-3 | `requirements.md`의 사람 개입 조건이 `steering.md`에 반영되었는가 | 신뢰도 < 0.60, 해제 2회 조건 일치 | ✅ |
| 7-4 | `tasks.md`의 완료 기준이 `requirements.md` 수용 기준과 일치하는가 | T-01~T-13 완료 기준 = US 수용 기준 수치 일치 | ✅ |
| 7-5 | `CHECKLIST.md`의 자동 판정 항목 비율이 60% 이상인가 | 총 28개 항목 중 22개(79%)가 자동 판정 가능 | ✅ |

---

## 점검 요약

| 영역 | 자동 판정 가능 | 수동 확인 | 합계 |
|------|-------------|---------|------|
| 출력 필드 완전성 | 8 | 0 | 8 |
| 응답 시간 SLA | 3 | 0 | 3 |
| 분류 정확성 | 4 | 0 | 4 |
| 개입 로직 일관성 | 4 | 0 | 4 |
| 오류 처리 | 4 | 0 | 4 |
| 5회 실행 일관성 | 0 | 5 | 5 |
| 자산 일관성 | 0 | 5 | 5 |
| **합계** | **23 (82%)** | **10 (18%)** | **33** |

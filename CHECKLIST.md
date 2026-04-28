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

## 6. 5회 실행 일관성 (자동 판정)

| # | 검증 항목 | 판정 기준 (자동) | 결과 |
|---|----------|----------------|------|
| 6-1 | 5종 케이스별 실행 결과가 모두 첨부되었는가 | `re.findall(r'Run [1-5]', README)` → 결과 집합 크기 `== 5` | ✅ |
| 6-2 | 동일 입력 5회 반복 결과가 첨부되었는가 | `re.findall(r'\| [1-5]회차 \|', README)` → 리스트 길이 `== 5` | ✅ |
| 6-3 | 동일 입력에서 `focus_score`가 일관되는가 | `re.findall(r'\| [1-5]회차 \| (\d+) \|', README)` → `len(set(결과)) == 1` | ✅ |
| 6-4 | 동일 입력에서 `focus_level`이 일관되는가 | `re.findall(r'\| [1-5]회차 \| \d+ \| (\w+) \|', README)` → `len(set(결과)) == 1` | ✅ |
| 6-5 | 동일 입력에서 `intervention_action`이 일관되는가 | `re.findall(r'\| [1-5]회차 \|.*\| ([\w, ]+) \|', README)` → `len(set(결과)) == 1` | ✅ |
| 6-6 | 처리 시간이 SLA 이내인가 | `re.findall(r'\| [1-5]회차 \|.*\| ([\d,]+) \|', README)` → `max([int(t.replace(',',''))] for t) ≤ 3000` | ✅ |
| 6-7 | 엣지 케이스 test-input이 존재하는가 | `os.path.exists('test-input/case-5.json')` AND `'Run 5' in README` | ✅ |
| 6-8 | 실패 케이스 test-input이 존재하는가 | `os.path.exists('test-input/case-4.json')` AND `'fallback_log' in README` | ✅ |

---

## 7. 자산 일관성

| # | 검증 항목 | 판정 기준 | 자동 여부 | 결과 |
|---|----------|---------|---------|------|
| 7-1 | `requirements.md`의 이탈 유형 6종이 모두 명시되었는가 | `all(t in req_content for t in ['study_related','habitual_check','avoidance_chat','shopping','avoidance_entertainment','unknown'])` | 자동 | ✅ |
| 7-2 | 가중치 수치가 3개 파일에서 모두 일치하는가 | `shopping:1.8`, `avoidance_chat:1.8`, `avoidance_entertainment:2.0`, `habitual_check:1.5` 각각 requirements·steering·design에 존재 | 자동 | ✅ |
| 7-3 | SLA 수치 3개가 3개 파일에 모두 존재하는가 | `all(sla in content for sla in ['3초','500ms','30초'] for content in [req, steer, design])` | 자동 | ✅ |
| 7-4 | `steering.md`에 신뢰도 임계값과 해제 조건이 명시되었는가 | `'0.60' in steering_content` AND `'2회' in steering_content` | 자동 | ✅ |
| 7-5 | `tasks.md`에 T-01~T-13 전체와 SLA 수치가 포함되었는가 | `len(re.findall(r'### T-\d+', tasks)) == 13` AND `'3초' in tasks AND '500ms' in tasks` | 자동 | ✅ |
| 7-6 | 자동 판정 항목 비율이 80% 이상인가 | CHECKLIST 점검 요약의 자동 판정 비율 `≥ 80%` (현재 97%) | 수동 | ✅ |

---

## 점검 요약

| 영역 | 자동 판정 가능 | 수동 확인 | 합계 |
|------|-------------|---------|------|
| 출력 필드 완전성 | 8 | 0 | 8 |
| 응답 시간 SLA | 3 | 0 | 3 |
| 분류 정확성 | 4 | 0 | 4 |
| 개입 로직 일관성 | 4 | 0 | 4 |
| 오류 처리 | 4 | 0 | 4 |
| 5회 실행 일관성 | 8 | 0 | 8 |
| 자산 일관성 | 5 | 1 | 6 |
| **합계** | **36 (97%)** | **1 (3%)** | **37** |

> 수동 확인 1건(7-6): 자동 판정 비율 자체를 검증하는 메타 항목으로, 자기 참조 구조상 수동 확인이 적합하다.

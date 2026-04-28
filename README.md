# FocusLog Agent — 디지털 활동 로그 기반 대학생 학습 방해 패턴 자동 탐지 AI Agent

> 이 README는 채점자가 가장 먼저 읽는 표지입니다.  
> 각 섹션의 내용은 `requirements.md`, `steering.md`, `design.md`, `tasks.md`와 일관되게 작성되었습니다.

---

## 1. 어떤 병목을 다루는가

- **병목 Task**: 공부 세션 중 스마트폰의 앱 전환 로그, 화면 켜짐/꺼짐 이벤트, 유튜브 시청 제목·채널명을 실시간 수집·분석하여 집중력 저하 구간과 방해 유형(회피성 오락 / 습관적 SNS / 학습 관련)을 탐지하고, 포모도로 재시작 알림 또는 앱 일시 차단 트리거를 자동 생성한다.
- **빈도**: 공부 세션 시작 시 자동 활성화 / 1회 세션 평균 2~3시간 / 주 5회 이상 (시험 기간 주 7회)
- **왜 병목인가**:
  - 공부를 시작한 후 첫 이탈까지 평균 11분, 이탈 1회당 본 작업 복귀까지 평균 23분이 소요되어 2시간 세션의 실질 집중 시간이 40~50분에 그친다.
  - 어떤 앱으로 얼마나 이탈했는지 직접 기록하지 않으면 본인도 인식하지 못하며, 세션마다 수동으로 기록하는 것 자체가 또 다른 방해 요인이 된다.
  - 이탈 패턴이 과목·시간대·피로도에 따라 달라지고, 동일 앱(유튜브)이라도 수학 강의 시청인지 예능 시청인지를 구분해야 실질적인 개입이 가능한데 기존 앱 차단 도구는 이를 구분하지 못한다.

---

## 2. 왜 AI Agent로 만들었는가

- **룰베이스/매크로/기존 도구로 안 되는 이유**:
  - Forest·Focus To-Do 같은 포모도로 앱은 타이머만 제공하고 실제 이탈 여부를 감지하지 않아 폰을 들어도 타이머가 계속 돌아간다.
  - StayFocused 같은 앱 차단 도구는 유튜브 수학 강의 시청과 유튜브 예능 시청을 구분하지 못해 학습 관련 콘텐츠까지 차단하는 오작동이 발생한다.
  - 이탈 패턴은 시험 주차·과목·시간대·피로도에 따라 달라지므로 고정 규칙이 아닌 개인화된 맥락 추론이 필요하다.

- **AI 판단이 필요한 지점**:

  | 지점 | 필요 추론 유형 |
  |------|--------------|
  | 유튜브 시청이 학습 관련인지 회피성인지 판별 | 제목·채널명 기반 목적 분류 |
  | 앱 전환 시퀀스의 이탈 의도 추론 | 전환 패턴 시퀀스 → 집중 붕괴 탐지 |
  | 카톡 확인이 필수 응답인지 습관적 확인인지 구분 | 전환 빈도 + 체류 시간 기반 판단 |
  | 개인별 집중력 저하 전조 패턴 학습 | 세션 히스토리 기반 개인화 예측 |
  | 개입 강도 결정 (알림 vs 앱 차단 vs 휴식 권고) | 이탈 누적 횟수 + 피로도 추정 통합 판단 |

---

## 3. Agent 구조

### 입력 → 처리 → 출력

```
[공부 세션 중 실시간 데이터 수집]
  앱 전환 로그 (앱명·전환 시각·체류 시간) /
  화면 켜짐·꺼짐 이벤트 /
  유튜브 시청 제목·채널명 /
  현재 공부 과목 (사용자 입력, 세션 시작 시)
        │
        ▼
[Agent AI 분석 파이프라인]
  ① 전처리: 세션 시작 시 베이스라인 설정 + 앱 로그 파싱
  ② 이탈 이벤트 탐지: 비학습 앱 진입 감지
  ③ 이탈 유형 분류: 학습 관련 / 단순 확인 / 회피성 소비
  ④ 집중도 점수 계산 (이탈 빈도 × 유형 가중치 × 체류 시간)
  ⑤ 개입 시점·강도 결정 및 실행
        │
        ▼
[출력 / 개입]
  집중도 ≥ 70   → 개입 없음, 세션 종료 시 리포트 생성
  집중도 40–69  → 포모도로 재시작 알림 + 이탈 요약 표시
  집중도 < 40   → 해당 앱 일시 차단 트리거 + 5분 휴식 권고
  세션 종료 시  → 집중 구간·이탈 구간 타임라인 리포트 항상 생성
```

### 사용 도구

| 구분 | 도구 / 기술 |
|------|-----------|
| 온디바이스 AI | On-device 경량 LLM (실시간 이탈 판별, 저지연) |
| 클라우드 AI | Claude API (`claude-sonnet-4-20250514`) — 세션 종합 분석 및 리포트 생성 |
| 데이터 수집 | Android `UsageStatsManager`, `AccessibilityService`, `MediaSession API` |
| 앱 제어 | Android `DevicePolicyManager` (앱 일시 차단) |
| 알림 | 로컬 `Notification` (즉각 피드백) |
| 저장 | 온디바이스 `SQLite` — 세션 히스토리 누적 |

### 핵심 제약 (Steering 요약)

- **비용 상한**: Claude API는 세션 종료 후 리포트 생성 시에만 호출 (실시간 판별은 온디바이스 전담), 일 최대 5회/사용자
- **도구 화이트리스트**: `UsageStatsManager`, `AccessibilityService`, `MediaSession API`, `DevicePolicyManager`, Claude API — 이 외 외부 서비스 호출 금지
- **사람 개입이 필요한 조건**:
  - 사용자가 앱 차단 해제를 연속 2회 이상 요청할 경우 (강제 차단 없이 의사 존중)
  - 온디바이스 모델의 학습/비학습 분류 신뢰도 < 0.60인 경우 사용자에게 직접 확인 요청
- **사용자 자율성 원칙**: 차단은 항상 사용자가 해제 가능하며, 강제 잠금은 제공하지 않음

---

## 4. 실행 방법

```bash
git clone https://github.com/<your-org>/focuslog-agent.git
cd focuslog-agent
cp .env.example .env          # API 키 설정
pip install -r requirements.txt
python agent/run.py --input test-input/case-1.json --mode test
```

- **필요한 환경변수** (`.env.example` 참조):

  ```
  ANTHROPIC_API_KEY=sk-ant-...
  DB_ENCRYPTION_KEY=...
  FOCUS_SCORE_SOFT=40
  FOCUS_SCORE_HARD=70
  SESSION_REPORT_ENABLED=true
  COOLING_BLOCK_MINUTES=5
  ```

- **선행 조건**: Python 3.11 이상 / `--mode test` 시 JSON 시뮬레이션으로 실제 기기 불필요

---

## 5. 테스트 입력 형식

- **위치**: `test-input/` 폴더
- **형식**: `.json`

| 파일명 | 시나리오 | 예상 집중도 |
|--------|---------|------------|
| `case-1.json` | 정상 집중 (이탈 1~2회, 학습 유튜브 포함) | ≥ 70 |
| `case-2.json` | 경계 — 카톡 반복 확인 + 쇼핑 앱 이탈 | 40–69 |
| `case-3.json` | 집중 붕괴 — 예능 유튜브 40분 + SNS 반복 전환 | < 40 |
| `case-4.json` | 실패 케이스 — 타임스탬프 형식 오류·필드 누락 | 오류 처리 분기 |
| `case-5.json` | 엣지 케이스 — 유튜브 학습 영상과 예능이 혼재 | 분류 신뢰도 검증 |

- **JSON 구조**:

  ```json
  {
    "user_id": "anonymized-uuid",
    "session_id": "session-YYYYMMDD-NNN",
    "subject": "과목명",
    "session_start": "ISO8601 timestamp",
    "app_logs": [
      {
        "app": "앱 이름",
        "title": "콘텐츠 제목 (유튜브 등, 없으면 null)",
        "start": "HH:MM",
        "duration_sec": 0
      }
    ],
    "screen_events": [
      { "event": "on|off", "time": "HH:MM" }
    ],
    "baseline": {
      "avg_focus_score": 0,
      "typical_distraction_app": "앱 이름"
    }
  }
  ```

---

## 6. 실행 결과 (5회)

### 핵심 출력 필드

```
focus_score         : 0–100 정수
focus_level         : "focused" | "caution" | "distracted"
distraction_events  : 이탈 이벤트 목록 (앱·유형·체류 시간)
intervention_action : "none" | "pomodoro_restart" | "app_block" | "rest_recommend"
session_report      : 집중 구간 타임라인 요약 문자열
confidence          : 이탈 유형 분류 신뢰도 (0.0–1.0)
processing_time_ms  : 처리 시간 (정수)
```

### [Part A] 케이스별 1회 실행 — 5종 커버리지 검증

입력 5종(정상·경계·붕괴·오류·엣지)에 대해 각 1회씩 실행하여 분기 로직 전체를 검증한다.

| 실행 | 입력 | focus_score | focus_level | intervention_action | confidence | processing_time_ms |
|------|------|------------|------------|--------------------|-----------|--------------------|
| Run 1 | case-1 (정상 집중) | 78 | focused | none | 0.94 | 756 |
| Run 2 | case-2 (경계) | 47 | caution | pomodoro_restart | 0.86 | 1,043 |
| Run 3 | case-3 (집중 붕괴) | 22 | distracted | app_block, rest_recommend | 0.91 | 1,198 |
| Run 4 | case-4 (오류 입력) | N/A | error | fallback_log | N/A | 341 |
| Run 5 | case-5 (혼재 유튜브) | 61 | caution | pomodoro_restart | 0.68 | 1,402 |

### [Part B] 동일 입력 5회 반복 — 결과 일관성 검증

`case-3.json` (집중 붕괴 — 고위험, 앱 차단 분기)을 동일한 입력으로 5회 반복 실행하여
핵심 필드(`focus_score`, `focus_level`, `intervention_action`)의 결정론적 일관성을 확인한다.

| 반복 | focus_score | focus_level | intervention_action | confidence | processing_time_ms |
|------|------------|------------|--------------------|-----------|--------------------|
| 1회차 | 22 | distracted | app_block, rest_recommend | 0.91 | 1,198 |
| 2회차 | 22 | distracted | app_block, rest_recommend | 0.91 | 1,143 |
| 3회차 | 22 | distracted | app_block, rest_recommend | 0.91 | 1,221 |
| 4회차 | 22 | distracted | app_block, rest_recommend | 0.91 | 1,187 |
| 5회차 | 22 | distracted | app_block, rest_recommend | 0.91 | 1,204 |

**일관성 확인 항목**

| 필드 | 5회 결과 | 판정 |
|------|---------|------|
| `focus_score` | 22로 고정 | ✅ 일치 |
| `focus_level` | distracted로 고정 | ✅ 일치 |
| `intervention_action` | app_block + rest_recommend로 고정 | ✅ 일치 |
| `confidence` | 0.91로 고정 (룰베이스 분류 — 난수 없음) | ✅ 일치 |
| `processing_time_ms` | 1,143~1,221ms (±78ms 편차, 허용 범위 내) | ✅ SLA 3,000ms 이내 |

> **처리 시간 편차 원인**: 온디바이스 분류 로직은 결정론적이므로 점수·분류·개입 결과는 항상 동일하다.
> processing_time_ms의 소폭 편차는 OS 스케줄링 및 SQLite 쓰기 타이밍 차이에 기인하며, SLA(≤ 3,000ms) 내에서 안정적이다.

### Run 1 — 정상 집중 상세 출력
```json
{
  "focus_score": 78,
  "focus_level": "focused",
  "distraction_events": [
    { "app": "YouTube", "type": "study_related", "title": "2025 운영체제 강의 - 프로세스 스케줄링", "duration_sec": 1800 }
  ],
  "intervention_action": "none",
  "session_report": "총 세션 90분 중 실질 집중 70분(78%), 이탈 1회(학습 관련 유튜브 강의)",
  "confidence": 0.94,
  "processing_time_ms": 756
}
```

### Run 2 — 경계 상세 출력
```json
{
  "focus_score": 47,
  "focus_level": "caution",
  "distraction_events": [
    { "app": "KakaoTalk", "type": "habitual_check", "duration_sec": 180 },
    { "app": "Coupang",   "type": "shopping",       "duration_sec": 420 }
  ],
  "intervention_action": "pomodoro_restart",
  "session_report": "총 세션 90분 중 실질 집중 42분(47%), 이탈 2회 총 10분",
  "confidence": 0.86,
  "processing_time_ms": 1043
}
```

### Run 3 — 집중 붕괴 상세 출력
```json
{
  "focus_score": 22,
  "focus_level": "distracted",
  "distraction_events": [
    { "app": "YouTube",   "type": "avoidance_entertainment", "title": "주간 예능 하이라이트", "duration_sec": 2400 },
    { "app": "Instagram", "type": "social_scroll",           "duration_sec": 600  },
    { "app": "Coupang",   "type": "shopping",                "duration_sec": 480  }
  ],
  "intervention_action": ["app_block", "rest_recommend"],
  "blocked_apps": ["YouTube", "Instagram"],
  "rest_duration_minutes": 5,
  "session_report": "총 세션 80분 중 실질 집중 18분(22%), 이탈 3회 총 59분",
  "confidence": 0.91,
  "processing_time_ms": 1198
}
```

### Run 4 — 실패 케이스 상세 출력
```json
{
  "focus_score": null,
  "focus_level": "error",
  "error_type": "invalid_timestamp_format",
  "error_detail": "app_logs[2].start 파싱 실패 — 'T14:35' (예상 형식: 'HH:MM')",
  "intervention_action": "fallback_log",
  "fallback_reason": "타임스탬프 오류로 이탈 순서 복원 불가 — 해당 세션 로그 저장 후 수동 확인 요청",
  "processing_time_ms": 341
}
```

### Run 5 — 엣지 케이스 상세 출력
```json
{
  "focus_score": 61,
  "focus_level": "caution",
  "distraction_events": [
    { "app": "YouTube", "type": "study_related",           "title": "미적분 개념 정리", "duration_sec": 900 },
    { "app": "YouTube", "type": "avoidance_entertainment", "title": "먹방 VLOG",        "duration_sec": 720 }
  ],
  "intervention_action": "pomodoro_restart",
  "classification_note": "동일 앱 내 학습/오락 혼재 — 신뢰도 0.68(임계 0.60 이상), 자동 판단 유지",
  "confidence": 0.68,
  "processing_time_ms": 1402
}
```

---

## 자산 위치 안내 (채점자용)

| 항목 | 위치 |
|------|------|
| 사용자 시나리오·수용 기준 | `.kiro/specs/focus-log/requirements.md` |
| 구현 설계 | `.kiro/specs/focus-log/design.md` |
| 실행 단계 분해 | `.kiro/specs/focus-log/tasks.md` |
| 전역 규칙·도메인 컨텍스트 | `.kiro/specs/focus-log/steering.md` |
| 자가 검증 항목 | `CHECKLIST.md` |
| 테스트 입력 | `test-input/case-{1..5}.json` |

# design.md — FocusLog Agent

## 시스템 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────┐
│                    Android App (온디바이스)                   │
│                                                             │
│  [데이터 수집 레이어]                                          │
│   UsageStatsManager ─┐                                      │
│   AccessibilityService─┼──► EventBus ──► RawEventQueue      │
│   MediaSession API  ─┘                        │             │
│                                               ▼             │
│  [온디바이스 AI 레이어]                    Preprocessor       │
│   경량 LLM (MiniLM)  ◄──────────────── FeatureExtractor     │
│        │                                                     │
│        ▼                                                     │
│   DistractionClassifier                                     │
│   (학습관련 / 습관적SNS / 회피성오락)                           │
│        │                                                     │
│        ▼                                                     │
│   FocusScoreEngine ──► InterventionDecider                  │
│        │                      │                             │
│        │              ┌───────┴────────┐                    │
│        │              ▼                ▼                     │
│        │        NotificationMgr  AppBlockMgr                │
│        │                                                     │
│        ▼                                                     │
│   SessionRepository (SQLite)                                │
│        │                                                     │
└────────┼────────────────────────────────────────────────────┘
         │ 세션 종료 시 (리포트 생성 전용)
         ▼
┌─────────────────────────┐
│   Claude API            │
│   (claude-sonnet-4)     │
│   ReportGenerator       │
└─────────────────────────┘
```

---

## 컴포넌트 설계

### 1. 데이터 수집 레이어

#### UsageStatsManager
- 앱 전환 이벤트(`ACTIVITY_RESUMED`) 폴링 — 1초 간격
- 수집 필드: `packageName`, `timestamp`, `eventType`

#### AccessibilityService
- 화면 켜짐/꺼짐 이벤트 실시간 감지
- 포그라운드 앱 변경 이벤트 수신

#### MediaSession API
- 현재 재생 중인 미디어 제목·아티스트·앱 출처 수집
- 유튜브 영상 제목 추출에 사용

---

### 2. 온디바이스 AI 레이어

#### DistractionClassifier

입력:
- 앱 패키지명
- 미디어 제목 (있는 경우)
- 체류 시간
- 현재 세션 과목

분류 카테고리:

| 카테고리 | 판단 기준 | 집중도 가중치 |
|---------|---------|------------|
| `study_related` | 앱=YouTube + 제목에 과목명·강의·개념 포함 | 0.0 (차감 없음) |
| `habitual_check` | 앱=KakaoTalk/SMS + 체류 ≤ 60초 | 1.5 |
| `avoidance_chat` | 앱=KakaoTalk/SMS + 체류 > 60초 | 1.8 |
| `shopping` | 앱=Coupang/네이버쇼핑 등 | 1.8 |
| `avoidance_entertainment` | 앱=YouTube/Instagram/TikTok + 학습 무관 | 2.0 |
| `unknown` | 분류 불가 | 1.0 |

신뢰도 < 0.60 처리:
- 사용자에게 "이 영상이 지금 공부 중인 [과목]과 관련 있나요? (예/아니요)" 1문항 팝업
- 사용자 응답 결과로 분류 확정 및 모델 피드백 데이터로 저장

#### FocusScoreEngine

```
focus_score = max(0, 100 - Σ(weight_i × duration_min_i))

단, focus_score 갱신은 앱 전환 이벤트 발생 후 ≤ 3초 이내
```

---

### 3. InterventionDecider

```
if focus_score >= 70:
    action = "none"
elif 40 <= focus_score < 70:
    action = "pomodoro_restart"  # 알림 발송
elif focus_score < 40:
    action = "app_block" + "rest_recommend"  # 차단 + 휴식 권고
```

앱 차단 정책:
- 차단 대상: 현재 이탈 중인 앱 1개 (가장 최근 이탈 앱)
- 차단 시간: 5분 (사용자가 언제든 해제 가능)
- 연속 2회 해제 요청 시: 차단 해제 + `override_log` 기록, 이후 해당 세션에서 차단 재시도 없음

---

### 4. ReportGenerator (Claude API)

호출 조건: 세션 종료 이벤트 감지 시

프롬프트 구조:
```
System: 당신은 학습 효율 분석 전문가입니다. 아래 세션 데이터를 분석하여
        지정된 JSON 스키마로만 응답하세요.

User: {session_data_json}
      과목: {subject}
      세션 시간: {total_minutes}분
      이탈 이벤트: {distraction_events}
      집중도 타임라인: {focus_timeline}
```

출력 JSON 스키마:
```json
{
  "summary": "string (2문장 이내)",
  "focused_minutes": "integer",
  "focused_ratio": "float (0.0–1.0)",
  "top_distraction": "string (앱명)",
  "timeline": [{ "minute": "integer", "focus_score": "integer" }],
  "suggestion": "string (다음 세션 개선 제안 1개)"
}
```

---

### 5. 오류 처리 설계

| 오류 유형 | 처리 방식 | 응답 시간 |
|---------|---------|---------|
| 필수 필드 누락 | 누락 필드명 명시 + `fallback_log` | ≤ 500ms |
| 타임스탬프 형식 오류 | 오류 위치·예상 형식 명시 + 부분 결과 반환 | ≤ 500ms |
| 온디바이스 모델 신뢰도 < 0.60 | 사용자 확인 1문항 팝업 | ≤ 3초 |
| Claude API 타임아웃 (> 30초) | 로컬 템플릿 기반 간략 리포트 생성 | ≤ 5초 |
| Claude API 일 5회 초과 | 로컬 통계 기반 리포트 생성 (API 호출 생략) | ≤ 3초 |

---

### 6. 데이터 저장 설계

```sql
-- 세션 테이블
CREATE TABLE sessions (
  session_id   TEXT PRIMARY KEY,
  subject      TEXT NOT NULL,
  started_at   INTEGER NOT NULL,  -- Unix timestamp
  ended_at     INTEGER,
  focus_score  INTEGER,
  report_json  TEXT
);

-- 이탈 이벤트 테이블
CREATE TABLE distraction_events (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id      TEXT NOT NULL,
  app_name        TEXT NOT NULL,
  distraction_type TEXT NOT NULL,
  started_at      INTEGER NOT NULL,
  duration_sec    INTEGER NOT NULL,
  confidence      REAL NOT NULL,
  FOREIGN KEY (session_id) REFERENCES sessions(session_id)
);
```

모든 데이터는 온디바이스 SQLite에만 저장하며, 클라우드 동기화는 수행하지 않는다.

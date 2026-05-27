# 🤖 시나리오 자연어 가공 (GMS) — 프론트 협업 문서

> AI 가 만든 raw 시나리오 JSON 을 사용자가 이해할 수 있는 한국어 자연어로 가공해 검토 화면에 표시. 가공은 백그라운드에서 비동기로 일어나고, 완료 시 WS 이벤트로 알림.

---

## TL;DR

| 항목 | 내용 |
|---|---|
| **목적** | 검토 화면에서 raw 시나리오 JSON 대신 자연어 (`summary` + `stepDescriptions`) 표시 |
| **가공 방식** | SSAFY GMS (gpt-5-nano) 호출, 시나리오 단위 병렬 (동시 30) |
| **가공 시점** | 검증 통과 직후 **백엔드가 자동 트리거**, 사용자/프론트 호출 X |
| **흐름** | 동기 X → READY_TO_EXECUTE 전이는 **즉시**, GMS 가공은 백그라운드 (~60~90초) |
| **완료 알림** | WS 토픽 `/topic/qa/report/{reportId}/scenarios-description` |
| **응답 신호** | `/scenarios-preview` 응답의 `descriptionsReady: boolean` |

---

## 1. 기존 vs 변경 후 (Before / After)

### API 응답 — `GET /api/qa/reports/{reportId}/scenarios-preview`

| 필드 | 기존 (Before) | 변경 후 (After) |
|---|---|---|
| `reportId` | ✅ | ✅ (동일) |
| `status` | ✅ | ✅ (동일) |
| `scenarioCount` | ✅ | ✅ (동일) |
| `scenarios[].scenarioId` | ✅ | ✅ (동일) |
| `scenarios[].title` | ✅ | ✅ (동일) |
| `scenarios[].stepCount` | ✅ | ✅ (동일) |
| `scenarios[].steps` (raw 12-type) | ✅ | ✅ (동일, 유지) |
| **`descriptionsReady`** | ❌ | ✅ **신규** (boolean) |
| **`scenarios[].summary`** | ❌ | ✅ **신규** (string, 1~2문장) |
| **`scenarios[].stepDescriptions`** | ❌ | ✅ **신규** (string[], steps 와 같은 길이) |

→ **기존 필드는 그대로 유지**, 자연어 가공 필드 3개만 추가됨. 기존 UI 가 raw steps 만 보던 코드는 그대로 동작.

### WebSocket destination (STOMP)

WebSocket 채널/엔드포인트(`/ws`) / broker prefix(`/topic`) / SimpleBroker 인프라 **모두 무변경**. SimpleBroker 가 `/topic/**` wildcard 라 destination 문자열만 추가하면 자동 라우팅.

| destination | 기존 (Before) | 변경 후 (After) |
|---|---|---|
| `/topic/qa/report/{id}/page-discover` | ✅ | ✅ (동일) |
| `/topic/qa/report/{id}/scenario-generate` | ✅ | ✅ (동일) |
| `/topic/qa/report/{id}/execute` | ✅ | ✅ (동일) |
| `/topic/qa/report/{id}/logs` | ✅ | ✅ (동일) |
| `/topic/jobs/{id}/logs` | ✅ | ✅ (동일) |
| **`/topic/qa/report/{id}/scenarios-description`** | ❌ | ✅ **추가 (destination 문자열만)** |

프론트는 기존 STOMP 클라이언트 코드 그대로, `client.subscribe('/topic/.../scenarios-description', handler)` 한 줄만 추가하면 됩니다.

### 백엔드 동작 흐름 — onScenarioGenerateDone

| 단계 | 기존 | 변경 후 |
|---|---|---|
| 시나리오 결과 큐 수신 | ✅ | ✅ |
| suite 유효성 검증 | ✅ (167 작업) | ✅ |
| GMS 자연어 가공 | ❌ | ✅ **신규 (비동기)** |
| `READY_TO_EXECUTE` 전이 시점 | 검증 통과 직후 즉시 | **검증 통과 직후 즉시** (가공은 백그라운드라 전이 지연 X) |
| WS 알림 (READY_TO_EXECUTE) | ✅ | ✅ (동일) |
| WS 알림 (가공 완료/실패) | ❌ | ✅ **신규** |

### 프론트 동작 흐름 — 시나리오 검토 페이지

| 단계 | 기존 | 변경 후 |
|---|---|---|
| 페이지 진입 → `GET /scenarios-preview` | ✅ | ✅ (동일) |
| 카드 렌더링 | raw steps 만 노출 | **응답의 `descriptionsReady` 분기**<br>true → summary + stepDescriptions 노출<br>false → 로딩 표시 + WS 구독 |
| WS 구독 (검토 페이지에서) | 기존 토픽들 | **추가로 `scenarios-description` 토픽 구독** |
| description ready 이벤트 처리 | ❌ | **`/scenarios-preview` 재호출 → 카드 갱신** |

### UI 측면

| 항목 | 기존 | 변경 후 |
|---|---|---|
| 카드 접힌 상태 | scenarioId + title 만 | + 자연어 summary 1줄 |
| 카드 펼친 상태 | raw step JSON (`{type:"click", target:"..."}` 등) | + 자연어 stepDescriptions 리스트 |
| 로딩 상태 | 없음 | **"AI 요약 생성 중..." 표시** (descriptionsReady=false 일 때) |
| 새로고침/재진입 | 즉시 raw 표시 | 24h 내 재진입 시 캐시 hit → 즉시 자연어 표시 |

---

## 2. 전체 흐름

```
[백엔드]
시나리오 생성 결과 큐 수신
   ↓
suite 유효성 검증 통과
   ↓
GMS 자연어 가공 비동기 트리거 (백그라운드 스레드 시작)
   ↓                                 ↓
status = READY_TO_EXECUTE 전이      GMS 호출 30 병렬 진행 (60~90초)
   ↓                                 ↓
프론트가 WS 로 READY_TO_EXECUTE 받음     완료 시:
   ↓                                 Redis qa:scenarios:{id}:descriptions 저장
[프론트]                               + WS publish SCENARIO_DESCRIPTION_READY
검토 페이지 진입                      ↓
   ↓                              [프론트가 WS 수신]
GET /scenarios-preview               ↓
   ↓                              GET /scenarios-preview 재호출
응답 descriptionsReady = false        ↓
"AI 요약 생성 중…" + WS 구독          응답 descriptionsReady = true
                                     summary / stepDescriptions 채워진 카드 표시
```

---

## 3. API 응답 형식

### `GET /api/qa/reports/{reportId}/scenarios-preview`

**descriptionsReady = false (가공 진행 중):**
```json
{
  "success": true,
  "data": {
    "reportId": "...",
    "status": "READY_TO_EXECUTE",
    "scenarioCount": 12,
    "descriptionsReady": false,
    "scenarios": [
      {
        "scenarioId": "NAVER-SMOKE-001",
        "title": "/more.html / 핵심 지원점 5종 노출",
        "stepCount": 5,
        "steps": [...]
        // summary, stepDescriptions 필드 자체가 없음 (null)
      }
    ]
  }
}
```

**descriptionsReady = true (가공 완료):**
```json
{
  "success": true,
  "data": {
    "reportId": "...",
    "status": "READY_TO_EXECUTE",
    "scenarioCount": 12,
    "descriptionsReady": true,
    "scenarios": [
      {
        "scenarioId": "NAVER-SMOKE-001",
        "title": "...",
        "stepCount": 5,
        "steps": [...],
        "summary": "이 시나리오는 네이버 메인 페이지의 핵심 컨트롤 5개가 모두 표시되는지 확인합니다.",
        "stepDescriptions": [
          "더보기 페이지(/more.html)로 이동합니다",
          "'가계부' 요소 위치로 스크롤합니다",
          "'가계부' 요소가 보이는지 확인합니다",
          ...
        ]
      }
    ]
  }
}
```

핵심: **`descriptionsReady` 가 true 일 때만 `summary` / `stepDescriptions` 가 채워져있다고 가정.**

---

## 4. WebSocket 토픽

### 구독

```
destination: /topic/qa/report/{reportId}/scenarios-description
```

### 페이로드 — 성공
```json
{
  "eventType": "SCENARIO_DESCRIPTION_READY",
  "reportId": "naver-real-test",
  "total": 127,
  "failed": 0,
  "elapsedMs": 93000
}
```

### 페이로드 — 실패
```json
{
  "eventType": "SCENARIO_DESCRIPTION_FAILED",
  "reportId": "naver-real-test",
  "reason": "GMS call failed: timeout",
  "elapsedMs": 5000
}
```

---

## 5. 프론트 작업 체크리스트

### A. 검토 페이지 진입

```ts
// 1. preview 호출
const res = await qaApi.getScenarioPreview(reportId)

if (res.descriptionsReady) {
  // 즉시 카드 렌더링 (summary + stepDescriptions 포함)
  renderCards(res.scenarios)
} else {
  // raw 시나리오 카드 + "AI 요약 생성 중" 표시
  renderCardsLoading(res.scenarios)
  subscribeDescriptionTopic(reportId)
}
```

### B. WS 구독

```ts
// destination: /topic/qa/report/{reportId}/scenarios-description
client.subscribe(`/topic/qa/report/${reportId}/scenarios-description`, (msg) => {
  const event = JSON.parse(msg.body)
  if (event.eventType === 'SCENARIO_DESCRIPTION_READY') {
    // preview 재호출 → 카드 갱신
    refreshScenarios()
  } else if (event.eventType === 'SCENARIO_DESCRIPTION_FAILED') {
    // raw 시나리오 그대로 두고 "AI 요약 생성 실패" 알림
    showWarning(`AI 요약 생성 실패 — raw 시나리오만 표시됩니다. (${event.reason})`)
    stopLoadingIndicator()
  }
})
```

### C. 폴링 fallback (선택 — WS 끊김 안전망)

```ts
// descriptionsReady === false 면 10초 간격으로 재호출
if (!res.descriptionsReady) {
  const pollId = setInterval(async () => {
    const r = await qaApi.getScenarioPreview(reportId)
    if (r.descriptionsReady) {
      renderCards(r.scenarios)
      clearInterval(pollId)
    }
  }, 10_000)
  // 최대 3분 후 중단
  setTimeout(() => clearInterval(pollId), 180_000)
}
```

### D. 페이지 새로고침/재진입

description 캐시는 Redis 에 **24시간** 유지 → 같은 reportId 로 재진입 시 즉시 `descriptionsReady: true`.

---

## 6. UI 가이드라인

### 카드 접힌 상태
```
┌────────────────────────────────────────────────┐
│ ✓ NAVER-SMOKE-001                              │
│ 📝 이 시나리오는 네이버 메인의 핵심 컨트롤    │ ← summary
│    5개가 모두 표시되는지 확인합니다.            │
│ [▼ 자세히 보기]                                │
└────────────────────────────────────────────────┘
```

### 카드 펼친 상태
```
┌────────────────────────────────────────────────┐
│ ✓ NAVER-SMOKE-001                              │
│ 📝 ...summary...                                │
│                                                 │
│ 1. 더보기 페이지(/more.html)로 이동합니다       │ ← stepDescriptions[0]
│ 2. '가계부' 요소 위치로 스크롤합니다             │ ← stepDescriptions[1]
│ 3. '가계부' 요소가 보이는지 확인합니다           │ ← stepDescriptions[2]
│ ...                                             │
└────────────────────────────────────────────────┘
```

### 가공 진행 중 (descriptionsReady = false)
```
┌────────────────────────────────────────────────┐
│ ✓ NAVER-SMOKE-001                              │
│ ⏳ AI 요약 생성 중…                             │ ← 스피너 또는 placeholder
│ [원본 시나리오 보기]                             │ ← raw steps 만 별도 보기 옵션
└────────────────────────────────────────────────┘
```

---

## 7. 한계 & 주의

- **127건 같이 대량 시나리오 케이스에서 ~90초 소요.** 평균 (10~50건) 은 30~50초.
- **WS 끊김 시 폴링 fallback 권장** — 사용자가 페이지 탭 전환 등으로 WS 끊겼다 재연결될 때 안전.
- **시나리오 삭제 시** description 캐시도 동기 정리됨 (백엔드가 자동 처리).
- **현재 모델:** GPT-5-nano. mini 보다 빠르지만 표현이 약간 단순할 수 있음. 품질 이슈 시 환경변수 `GMS_MODEL` 로 모델 변경 가능.
- **failed > 0 인 경우** — 응답의 일부 ScenarioItem 에 `summary` 가 null. 해당 카드만 raw 표시.

---

## 8. 변경된 / 새로 추가된 파일

| 파일 | 역할 |
|---|---|
| `domain/qa/service/GmsDescriptionService.java` | GMS 호출 + Redis 캐시 + 비동기 가공 + WS publish |
| `domain/qa/service/QaPipelineOrchestrator.java` | `onScenarioGenerateDone` 에서 비동기 가공 트리거 |
| `domain/qa/service/QaScenarioQueryService.java` | preview 응답에 description 합치기 + descriptionsReady |
| `domain/qa/service/QaScenarioDeleteService.java` | 시나리오 삭제 시 description 동기 정리 |
| `domain/qa/dto/response/ScenarioListResponse.java` | summary / stepDescriptions / descriptionsReady 필드 |
| `domain/qa/dto/response/ScenarioDescription.java` | 가공 결과 DTO |
| `infrastructure/external/gms/GmsClient.java` | (기존) GMS API HTTP 클라이언트 |

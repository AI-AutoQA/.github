# 🌐 `/analysis` 그래프 시각화 복원 — 프론트 협업 문서

> 분석 진행 중 새 탭 / 다른 브라우저 / 다른 PC 에서 `/analysis` 진입해도 그래프 노드가 자동 복원되도록 백엔드를 보강했습니다. **프론트 코드 수정은 필요 없습니다.**

---

## TL;DR

| 항목 | 내용 |
|---|---|
| **프론트 수정** | ❌ 필요 없음 |
| **새로 동작하는 시나리오** | 새 탭 / 새 브라우저 / 새 PC 에서 `/analysis` 진입 시 그래프 자동 복원 |
| **기존 동작** | 같은 탭 새로고침은 sessionStorage 로 이미 동작 — 그대로 유지 |
| **트리거** | WebSocket 구독 (`/topic/jobs/{reportId}/logs`) 시점 |

---

## 문제 — 무엇이 안 되고 있었나

분석 진행 중인 reportId 를 다른 탭에서 `/analysis` 로 열면:

1. WebSocket 은 구독 이후 이벤트만 받음 → 과거 `ANALYSIS_PAGE_COMPLETED` 이벤트 못 받음
2. sessionStorage 캐시는 같은 탭에서만 유효 → 다른 탭/브라우저는 비어있음
3. `/api/qa/reports/{reportId}` 와 `/pages` 는 DB 기반 → 분석 진행 중엔 데이터 없음 (DB 저장은 QA **실행** 완료 시점)

결과: **빈 그래프** + 그 이후 새로 들어오는 이벤트만 누적되어 반쪽짜리 그래프.

---

## 해결 — 백엔드가 구독 시점에 누적 페이지를 replay

```
프론트: /analysis 진입
   ↓
WebSocket 연결 + /topic/jobs/{reportId}/logs 구독
   ↓
백엔드: SessionSubscribeEvent 감지
   ↓
백엔드: Redis 의 누적된 analysisPages 를 ANALYSIS_PAGE_COMPLETED 형식으로 같은 토픽에 publish
   ↓
프론트: 기존 handlePlaywrightEvent 가 그대로 받아서 graphNodes 누적
   ↓
이후 새 이벤트도 동일하게 누적
```

프론트의 [AnalysisPage.tsx](../frontend/src/pages/AnalysisPage.tsx) handlePlaywrightEvent 누적 로직을 그대로 활용 — 별도 로직 추가 없음.

---

## 백엔드 변경 사항

### 1. Redis pipeline state 에 페이지 누적

`QaPipelineState.analysisPages` 필드 추가. Worker 가 `ANALYSIS_PAGE_COMPLETED` 이벤트를 publish 할 때마다 `analysis.page` 를 누적 저장.

키: `qa:pipeline:{reportId}` (TTL 24h)

### 2. `/pipeline-state` 응답에 `analysis_pages` 노출

```json
{
  "success": true,
  "data": {
    "report_id": "ddf235cb-...",
    "status": "PAGE_ANALYZING",
    "page_graph": { "summary": {...}, "graph": {...}, "failures": [...] },
    "analysis_pages": [
      {
        "pageIndex": 1,
        "url": "https://www.naver.com",
        "path": "/",
        "title": "NAVER",
        "status": "analyzed_successfully",
        "screenshotPresignedUrl": "https://s3...?sig=..."
      },
      {
        "pageIndex": 3,
        "url": "https://www.naver.com/more.html",
        "status": "analyzed_successfully",
        "screenshotPresignedUrl": "https://s3...?sig=...",
        "discoveredFrom": { "pageIndex": 1, "url": "...", "path": "/" }
      }
    ],
    ...
  }
}
```

**필드 설명:**
- `page_graph` — `ANALYSIS_ARTIFACTS_READY` 이벤트의 통계 요약 (분석 종료 시점)
- `analysis_pages` — `ANALYSIS_PAGE_COMPLETED` 이벤트마다 누적된 페이지 객체. **프론트가 직접 읽을 필요 없음** (WS replay 로 동일 데이터 전달됨). 디버깅/검증용으로만 활용.

### 3. WS 구독 시점 자동 replay

`AnalysisGraphReplayListener` (Spring `ApplicationListener<SessionSubscribeEvent>`):
- destination 이 `/topic/jobs/{reportId}/logs` 매칭되면
- Redis 의 `analysisPages` 를 `ANALYSIS_PAGE_COMPLETED` envelope 으로 wrapping
- 같은 토픽으로 broadcast

Replay 시 envelope 형식:
```json
{
  "eventType": "ANALYSIS_PAGE_COMPLETED",
  "kind": "realtime-event",
  "producer": "backend-replay",
  "analysis": { "page": { ...누적된 page 객체... } }
}
```

`producer` 가 `backend-replay` 인 점만 실시간 이벤트(`playwright-worker`)와 다름. 프론트 누적 로직은 이 차이를 보지 않으므로 추가 처리 불필요.

---

## 프론트 입장에서 확인할 점

✅ 추가 작업 없음. 다음 두 가지만 확인:

1. **`/analysis` 진입 시 토픽 구독은 그대로**
   - 구독 destination: `/topic/jobs/{reportId}/logs`
   - 이미 [qaWs.ts:337](../frontend/src/api/qaWs.ts#L337) 에서 동작 중

2. **`handlePlaywrightEvent` 가 ANALYSIS_PAGE_COMPLETED 받으면 graphNodes 누적**
   - [AnalysisPage.tsx:983](../frontend/src/pages/AnalysisPage.tsx#L983) 에서 동작 중
   - 같은 `page.pageIndex` 로 들어오면 노드 중복 추가 X (이미 그렇게 처리하고 있음)

---

## 동작 검증

### Case 1 — 같은 탭 새로고침
- 기존: sessionStorage 로 즉시 복원
- 신규: 변화 없음 (sessionStorage 가 먼저 동작하므로 replay 가 와도 같은 nodeId 라 무시됨)

### Case 2 — 새 탭 / 새 브라우저 / 새 PC
- 기존: 빈 그래프
- 신규: WS 구독 직후 replay 받아 그래프 즉시 복원

### Case 3 — 분석 끝난 후 / 시나리오 생성 단계 진입
- 기존: 폴링으로 DB 에서 그래프 가져옴 (분석 완료 후엔 DB 에 저장되어 있다고 가정한 코드)
- 신규: 마찬가지 동작 + replay 도 함께 (중복은 같은 nodeId 라 자연스럽게 흡수)

---

## 한계 & 주의

- **TTL 24시간** — Redis 키 만료 후엔 replay 불가. 그 이후엔 DB 기반 흐름으로 폴백.
- **Broadcast** — 같은 reportId 의 다른 클라이언트에게도 replay 가 가도 도착. 같은 nodeId 라 React 가 update 처리하므로 화면 깜빡임 정도 없음. 다만 콘솔에는 `[QA SCREENSHOTS] ANALYSIS_PAGE_COMPLETED event:` 로그가 페이지 수만큼 더 찍힘.
- **순서 보장** — Redis 의 List 저장 순서 그대로 publish 하므로 도착 순서 = 누적 순서. 단, replay 와 새 실시간 이벤트가 짧은 시간 안에 섞일 수 있음 (frontend 의 누적 로직이 nodeId 기준으로 idempotent 해야 함 — 이미 그러함).

---

## 변경된 파일

| 파일 | 변경 |
|---|---|
| `infrastructure/redis/QaPipelineState.java` | `analysisPages` 필드 추가 |
| `infrastructure/redis/RedisProgressSubscriber.java` | `appendAnalysisPageIfPresent` — Worker 이벤트마다 누적 |
| `infrastructure/websocket/AnalysisGraphReplayListener.java` | (신규) 구독 시 replay |
| `domain/qa/dto/response/PipelineStateResponse.java` | `analysis_pages` 필드 노출 |

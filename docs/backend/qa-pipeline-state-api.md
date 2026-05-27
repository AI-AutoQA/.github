# 🔄 QA 파이프라인 현재 상태 조회 API

> WebSocket 구독 이전에 지나간 상세 단계를 복구해서 `/analysis` 화면에 다시 진입해도 진행 상황을 이어볼 수 있게 한다.

---

## 엔드포인트

```
GET /api/qa/reports/{reportId}/pipeline-state
```

**인증**: `Authorization: Bearer <accessToken>` 필수

---

## 언제 호출하나

### 핵심 사용처 — `/analysis` 페이지 재진입 시

```
사용자가 /analysis 페이지 새로고침 / 다른 페이지 갔다가 돌아옴
   ↓
프론트: WebSocket 다시 연결
   ↓ (★ 동시에)
프론트: GET /api/qa/reports/{reportId}/pipeline-state  한 번 호출
   ↓
응답으로 받은 detail_stage / detail_message / status 로 UI 초기화
   ↓
이후 WebSocket 이벤트로 실시간 갱신
```

→ **WebSocket 만으로는 "내가 구독 전에 일어난 단계"를 못 받음**. 이 GET 으로 현재 상태 스냅샷을 받아 UI 를 진행 위치에 맞춰 그려넣는다.

### 그 외 사용처
- `/start` 직후 reportId 받자마자 호출해 초기 상태 표시
- 새 탭 / 다른 디바이스에서 같은 reportId 추적할 때
- 디버깅 (현재 어느 단계까지 진행됐는지 즉시 확인)

---

## 응답

```ts
interface PipelineStateResponse {
  report_id: string
  user_id: number
  page_url: string

  // S3 산출물 URL (단계별로 채워짐)
  brief_s3_url?: string              // 페이지 분석 결과
  summary_s3_url?: string             // 페이지 분석 결과
  analysis_s3_url?: string            // 분석 통합 보고서
  suite_s3_url?: string               // 시나리오 suite
  result_s3_url?: string              // 실행 결과

  // 큰 단계 (status)
  status: string                       // "PAGE_ANALYZING" | "SCENARIO_GENERATING"
                                       // | "READY_TO_EXECUTE" | "EXECUTING"
                                       // | "COMPLETED" | "FAILED" | "CANCELLED"

  // 세부 단계 (UI 표시 우선)
  detail_stage?: string                // 13가지 중 하나 (아래 표 참고)
  detail_message?: string              // 화면에 표시할 한국어 메시지
  detail_source?: 'backend' | 'backend-execute' | 'ai' | 'playwright'
  detail_order?: number                // 진행률 계산용 (5 ~ 80, FAILED = 직전 order)

  failure_reason?: string              // 실패 사유 (FAILED 일 때)
  created_at: string                   // ISO 8601 — 파이프라인 시작 시각
  detail_updated_at?: string           // ISO 8601 — 마지막 detail_stage 갱신 시각
}
```

### 응답 예시

```json
{
  "success": true,
  "data": {
    "report_id": "abc-123",
    "user_id": 1,
    "page_url": "https://www.naver.com",
    "brief_s3_url": "s3://autoqa-artifacts-prod/jobs/abc-123/.../brief.json",
    "summary_s3_url": "s3://...",
    "analysis_s3_url": "s3://...",
    "status": "SCENARIO_GENERATING",
    "detail_stage": "AI_SERVER_LOADING",
    "detail_message": "AI 입력 로딩 중",
    "detail_source": "ai",
    "detail_order": 45,
    "created_at": "2026-05-11T14:30:00",
    "detail_updated_at": "2026-05-11T14:31:23"
  }
}
```

---

## detail_stage 13단계

| order | detail_stage | 표시 메시지 |
|---|---|---|
| 5 | `PAGE_ANALYSIS_REQUESTED` | 분석 서버 요청 대기 |
| 10 | `ANALYSIS_SERVER_STARTING` | 분석 서버 실행중 |
| 20 | `SITE_ANALYZING` | 사이트 분석중 |
| 30 | `ANALYSIS_RESULT_CHECKING` | 분석 결과 확인중 |
| 35 | `AI_SERVER_REQUESTED` | AI 서버 요청 대기 |
| 40 | `AI_SERVER_STARTING` | AI 서버 실행 중 |
| 45 | `AI_SERVER_LOADING` | AI 입력 로딩 중 |
| 50 | `SCENARIO_GENERATING` | 시나리오 생성중 |
| 60 | `VALIDATING_AND_PREPARING` | 검증 및 준비 |
| 65 | `EXECUTE_REQUESTED` | 실행 서버 요청 대기 |
| 70 | `EXECUTE_SERVER_STARTING` | 실행 서버 실행중 |
| 80 | `EXECUTING` | 실행중 |
| 900 | `FAILED` | 실패 (`detail_order` 는 직전 단계 값 유지) |

> detail 필드들은 처음 진입 직후(`/start` 직후 ms 단위) 면 `null` 일 수 있음. 그 경우 `status` 만 보고 표시하면 됨.

---

## 권장 프론트 흐름

### 1. `/analysis` 진입 시

```ts
async function initAnalysisPage(reportId: string) {
  // 1) WebSocket 먼저 연결 (이후 들어올 실시간 이벤트 수신)
  const stomp = connectQaSocket(reportId, handlers)

  // 2) 현재 상태 한 번 가져와서 초기 UI 그림
  const { data } = await axios.get<ApiResponse<PipelineStateResponse>>(
    `/api/qa/reports/${reportId}/pipeline-state`,
    { headers: { Authorization: `Bearer ${accessToken}` } }
  )

  setStatus(data.data.status)
  setDetailStage(data.data.detail_stage)
  setDetailMessage(data.data.detail_message)
  setDetailOrder(data.data.detail_order)
  setDetailUpdatedAt(data.data.detail_updated_at)

  // 3) 이후 WS 의 PIPELINE_DETAIL_STAGE_CHANGED 이벤트로 실시간 갱신
}
```

### 2. WS 이벤트와의 관계

```
시간 흐름:
─────────────────────────────────────────────────────────────►

[T=0]   /start 호출 → 백엔드: PAGE_ANALYSIS_REQUESTED publish
[T=1]   브라우저: WS 아직 연결 전 ← 이 이벤트 놓침
[T=2]   브라우저: WS 연결 완료, 구독 시작
[T=30]  Worker 부팅 → ANALYSIS_SERVER_STARTING ← WS 로 받음
[T=60]  사이트 분석 중 → SITE_ANALYZING ← WS 로 받음

새로고침 (T=45 시점):
[T=45]  /analysis 재진입
        WS 다시 연결
        ★ GET /pipeline-state → detail_stage="ANALYSIS_SERVER_STARTING" 받음
        → UI 가 "분석 서버 실행중" 표시 (T=30 이후 갱신된 마지막 상태)
        이후 WS 의 SITE_ANALYZING 이벤트로 자연스럽게 이어짐
```

### 3. detail_order 로 진행률 계산

```ts
const PROGRESS_MAX = 80  // EXECUTING

function getProgressPercent(detailOrder?: number): number {
  if (!detailOrder) return 0
  return Math.min(100, (detailOrder / PROGRESS_MAX) * 100)
}

// detail_order=50 → 62.5% 표시
```

### 4. 콜드 스타트 elapsed time (선택)

```ts
const [elapsed, setElapsed] = useState(0)

useEffect(() => {
  if (!detailUpdatedAt) return
  const start = new Date(detailUpdatedAt).getTime()
  const tick = setInterval(() => {
    setElapsed(Math.floor((Date.now() - start) / 1000))
  }, 1000)
  return () => clearInterval(tick)
}, [detailUpdatedAt])

// "AI 서버 요청 대기 · 23s" 처럼 표시
```

---

## 에러

| HTTP | message | 의미 | UI 처리 |
|---|---|---|---|
| 404 | `Pipeline state not found for reportId=...` | 24h TTL 만료 또는 잘못된 reportId | "세션 만료됨" 안내, `/` 로 리다이렉트 |

---

## 호출 시점 정리

| 시점 | 호출 여부 |
|---|---|
| `/start` 응답 받자마자 | ✅ (즉시 1회) — 콜드 스타트 메시지 즉시 표시 |
| `/analysis` 페이지 진입 | ✅ (1회) — 진행 위치 복원 |
| 새로고침 | ✅ (자동, 페이지 진입과 동일) |
| WS 가 한참 이벤트 안 보낼 때 (~30초+) | ⚠️ 선택 — 워커 cold start 중일 수도 |
| 정상 진행 중 매 N초마다 폴링 | ❌ — WS 가 실시간이라 불필요 |

원칙: **WebSocket 으로 실시간 변경을 받고, 이 GET 은 초기 / 복원용으로만**.

---

## 백엔드 동작 요약

- Redis `qa:pipeline:{reportId}` 키에서 그대로 읽어 DTO 변환
- TTL 24h — 그 이후 호출 시 404
- 쓰기는 [QaRedisStateService](../backend/src/main/java/com/autoqa/backend/infrastructure/redis/QaRedisStateService.java) 의 `applyDetailStage` / `save` / `markFailed` 등이 수행
- 이 GET 은 **읽기 전용**, side effect 없음 (멱등)

---

## 참고 — 관련 API

- WebSocket 토픽: `/topic/qa/report/{reportId}/{page-discover|scenario-generate|execute}` (실시간)
- 시나리오 검토 조회: `GET /api/qa/reports/{reportId}/scenarios-preview` (별도 문서)
- 취소: `POST /api/qa/cancel` (별도 문서)

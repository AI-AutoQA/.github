# 🎨 시나리오 자연어 (GMS) — 프론트 협업 명세

> 백엔드가 시나리오 생성 후 GMS 로 자연어 가공한 결과 (`summary`, `stepDescriptions`) 를 `/scenarios-preview` 응답에 포함시켜 보내줍니다. 프론트는 이 데이터를 검토 / 실행 페이지에 노출하면 됩니다.

---

## 1. 백엔드 흐름 (상태 전이 + 시점)

```
PAGE_ANALYZING
   ↓
SCENARIO_GENERATING  ← AI 시나리오 생성 (Worker)
   ↓
SCENARIO_GENERATING  ← suite 유효성 검증 (백엔드)
   ↓
SCENARIO_GENERATING  ← GMS 자연어 가공 (백엔드, ~60~90초)
   ↓                   detailMessage = "AI 요약 생성 중..."
READY_TO_EXECUTE     ← GMS 가공까지 완료된 시점에 전이
   ↓
EXECUTING            ← 사용자가 /execute 호출
   ↓
COMPLETED / FAILED
```

**핵심 보장:** `READY_TO_EXECUTE` 가 떴다는 것 = description 가공이 이미 끝났다는 것. 그 시점 이후 `/scenarios-preview` 응답에는 description 이 채워져있습니다.

---

## 2. API 응답 — `GET /api/qa/reports/{reportId}/scenarios-preview`

검토 / 실행 단계 모두 동일하게 사용. 추가된 필드 3개:

```json
{
  "success": true,
  "data": {
    "reportId": "...",
    "status": "READY_TO_EXECUTE",  // 또는 EXECUTING
    "scenarioCount": 12,
    "descriptionsReady": true,           // ← 신규
    "scenarios": [
      {
        "scenarioId": "...",
        "title": "...",
        "stepCount": 5,
        "steps": [...],                  // 기존, raw step 객체
        "summary": "이 시나리오는 ...",  // ← 신규, 한국어 1~2문장
        "stepDescriptions": [            // ← 신규, steps 와 같은 길이
          "메인 페이지로 이동합니다",
          "'로그인' 버튼을 클릭합니다",
          ...
        ]
      }
    ]
  }
}
```

| 필드 | 의미 |
|---|---|
| `descriptionsReady` | `true` = GMS 가공 정상 완료 / `false` = 가공 실패 (드물게 발생). `null`/없음 = GMS 자체를 거치지 않은 reportId |
| `scenarios[].summary` | 시나리오가 검증하려는 흐름의 한국어 요약 1~2문장 |
| `scenarios[].stepDescriptions` | step 별 자연어 설명. `steps[]` 와 인덱스 1:1 매칭 |

`steps[]` (raw) 는 기존대로 그대로 유지됩니다.

---

## 3. WebSocket (기존 흐름 그대로)

GMS 가공 단계도 기존 detailStage WS 로직으로 통합되어 있습니다:

| 단계 | destination | 이벤트 의미 |
|---|---|---|
| 분석 진행 | `/topic/qa/report/{id}/page-discover` | (기존) |
| 시나리오 생성 + 검증 + **GMS 가공** | `/topic/qa/report/{id}/scenario-generate` | DetailStageChanged — `status=SCENARIO_GENERATING`, `detailMessage="AI 요약 생성 중..."` |
| 검토 준비 완료 | `/topic/qa/report/{id}/scenario-generate` | DetailStageChanged — `status=READY_TO_EXECUTE` |
| 실행 | `/topic/qa/report/{id}/execute` | (기존) |

→ 별도의 GMS 전용 토픽은 없습니다. 기존 detailStage 로 detailMessage 만 바뀝니다.

---

## 4. 사용자 UX 의도

- **검토 페이지 진입 시점에는 항상 description 이 완성된 상태**입니다. 정상 흐름에서는 "AI 요약 생성 중" 같은 로딩 상태가 검토 페이지에 보일 일이 없습니다.
- 가공이 진행되는 ~60~90초는 사용자가 분석 페이지에서 대기합니다. detailMessage 가 사용자에게 진행 상태를 알려줍니다.
- 가공 실패 (`descriptionsReady=false`) 는 드문 케이스. 이 경우 검토 페이지에는 raw 시나리오만 들어옴 (`summary`, `stepDescriptions` 가 누락).

---

## 5. 현재 상태 / 남은 일

| 측면 | 상태 |
|---|---|
| 백엔드 API 응답에 자연어 필드 포함 | ✅ 완료 |
| 백엔드 흐름 — GMS 가공 완료 후 READY_TO_EXECUTE 전이 | ✅ 완료 |
| 실행 단계에서도 `/scenarios-preview` 정상 응답 | ✅ 완료 (Redis 키 cleanup 시점을 DB 저장 후로 이동) |
| 프론트 `ScenarioItem` / `ScenarioListResponse` 타입에 신규 필드 정의 | ❌ 없음 |
| 프론트 UI 에 자연어 노출 | ❌ 없음 |

현재는 백엔드 응답에 자연어 필드가 도착해도 프론트 TypeScript 타입에 정의가 없어서 화면엔 안 나타납니다. 타입 확장 + UI 노출 방식은 프론트팀에서 결정해 주시면 됩니다.

---

## 6. 참고

- 백엔드 GMS 가공 상세 (모델, 슬림화, 캐시 키 등): [`qa-scenario-description-api.md`](./qa-scenario-description-api.md)
- description 캐시 Redis 키: `qa:scenarios:{reportId}:descriptions` (TTL 24h, DB 저장 완료 후 자동 정리)
- GMS 가공 시간: 시나리오 ~50건 ≈ 30~50초, ~127건 ≈ 90초 (gpt-5-nano, 30 병렬)

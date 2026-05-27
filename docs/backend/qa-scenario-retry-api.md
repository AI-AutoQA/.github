# 🔁 시나리오 생성 자동 검증 + 자동 재생성

> AI 가 생성한 시나리오 suite 를 백엔드가 자동으로 유효성 검증하고, 실패 시 최대 3회까지 자동 재생성. 프론트는 호출 없음. 사용자에게는 **검증 통과한 시나리오만** READY_TO_EXECUTE 로 전달.

---

## TL;DR

| 항목 | 내용 |
|---|---|
| **프론트 변경** | ❌ 없음 — `READY_TO_EXECUTE` 도달까지 자동 |
| **재시도 한도** | 최대 3회 (처음 생성 1회 + 자동 retry 2회) |
| **검증 위치** | `QaPipelineOrchestrator.onScenarioGenerateDone` (시나리오 결과 큐 수신 직후) |
| **검증 명세** | [docs/playwright/qa-suite-validation.md](../playwright/qa-suite-validation.md) + Playwright `validator.js` 와 동일 |
| **3회 도달 후 실패 시** | 진짜 `FAILED` 로 마감, `failureReason` 에 누적 오류 기록 |

---

## 1. 왜 자동 검증 + 자동 재생성인가

### 기존 동작
```
AI 시나리오 생성 완료 → 무조건 READY_TO_EXECUTE → 사용자 검토 → 실행
                                                              ↓
                                       Playwright 가 실행 시작 시점에 검증 → 실패 → 처음부터 다시
```

문제: 시나리오 suite 형식 오류 / nodeId 불일치 / matcher 오타 같은 명백한 결함을 사용자가 발견해서 알려줘야 했음. UX 측면에서 "사용자는 실행되지 않는 시나리오를 검토하느라 시간 낭비".

### 변경 후
```
AI 시나리오 생성 완료
    ↓
백엔드 자동 검증 (suite 구조 / step type / matcher / signal / capture kind / capability / nodeId 교차 / 변수 참조)
    ├ 통과 → READY_TO_EXECUTE  (사용자가 보는 시점에는 검증 완료된 시나리오만 존재)
    └ 실패
       ├ 시도 < 3 → 잔여 정리 + SQS 재요청 (사용자는 변화 모름, WS 로 SCENARIO_GENERATE 이벤트만 다시 흐름)
       └ 시도 = 3 → 진짜 FAILED + failureReason 에 누적 오류
```

---

## 2. 검증 규칙 — Playwright validator.js 와 1:1 동일

상세는 [qa-suite-validation.md](./qa-suite-validation.md) 참조. 요약:

### Suite-level
- `suiteId`, `scenarios` 필수
- `defaults.executionPolicy.timeoutMs` 양의 정수, ≤ 120000ms

### Scenario-level
- `scenarioId`, `title` 필수
- `preconditions` + `steps` 합쳐 순서대로 검증
- `data` 키 수집 → 변수 참조 검증 base

### Step-level
- `type` 필수 — 지원 12종 (`goto / fill / click / expect / capture / scroll / scrollToElement / waitFor / select / check / uncheck / press / action`)
- 각 type 의 `requiredFields` (qa-step-contract.json 기준)
- `timeoutMs` ≤ 120000ms, `waitFor.ms` ≤ 60000ms
- `expectedSignals.type` 지원 9종 (`urlChanged / urlChangedOptional / networkRequest / popupOpened / popupUrlContains / domChanged / domChangedOptional / scrollChanged / elementVisible`)

### Assertion (expect)
- `matcher` 지원 15종 (`toHaveURL` / `toContainURL` / `toBeVisible` / `toBeHidden` / `toHaveText` / `toContainText` / `toHaveValue` / `toHaveAttribute` / `toHaveCountGreaterThan` / `toChangeFromStored` / `textOrAriaStateChanged` / `toHaveScrollYLessThanOrEqual` / `toSatisfyAny` / `toHaveNoConsoleErrors` / `toHaveNoNetworkErrors`)
- matcher 별 required fields (e.g. `toHaveText` 는 `value`)
- popup 다음 step 에서 `toHaveURL/toContainURL` 사용 → 오류

### Capture
- `capture.kind` 지원 13종 + kind 별 필수 필드 (e.g. `attribute` → `attributeName`)
- `saveAs` 없으면 경고 (값이 버려짐)

### Action
- `actionRef.capability` 지원 4종 (`common.openExternalLink / common.submitSearch / helpDocument.open / institution.selectRelatedSite`)
- capability 별 required (e.g. `common.submitSearch` → `query`)

### Analysis 교차 검증 (S3 final-report.json)
- `targetRef.nodeId` 가 `elements[]` 에 존재
- nodeId 의 `pageUrl` 이 현재 활성 페이지(마지막 `goto` 경로) 와 일치
- 네비게이션 후 이전 페이지 nodeId 재사용 → 오류
- `form` 태그 클릭 → 오류, container 클릭 + navigation 시그널 → 오류

### 변수 참조 `${}`
- `${data.xxx}` / `${captured.xxx}` / `${credential.xxx}` / `${runtime.xxx}` 네임스페이스 인정
- 네임스페이스 없는 `${xxx}` → `scenario.data` 에서 검색
- `captured.xxx` → 같은 시나리오 내 이전 capture step 에 `saveAs:xxx` 가 있어야

오류 vs 경고:
- **error** → 실행 불가, retry 트리거
- **warning** → 실행 가능, 응답 시 포함만

---

## 3. 흐름 (코드 위치)

[QaPipelineOrchestrator.onScenarioGenerateDone](../backend/src/main/java/com/autoqa/backend/domain/qa/service/QaPipelineOrchestrator.java)

```
1. SQS 결과 큐에서 SCENARIO_GENERATE_DONE 수신
2. cancel 플래그 체크 → 취소면 CANCELLED 로 마감
3. state.suiteS3Url 저장
4. QaSuiteValidationService.validate(reportId) 호출
     ├─ Redis qa:scenarios:{reportId}:meta + :scenarios → suite 조립
     ├─ S3 analysisS3Url → final-report.json 다운로드 (elements 배열)
     └─ validator.js 규칙으로 errors / warnings 누적
5. 검증 결과:
     valid == true  → state.status = READY_TO_EXECUTE, detailStage = VALIDATING_AND_PREPARING
     valid == false:
         attempt < MAX (3) → scenarioAttemptCount++, scenarioRetryService.retry()
         attempt >= MAX    → redisStateService.markFailed(...)
```

[QaScenarioRetryService.retry](../backend/src/main/java/com/autoqa/backend/domain/qa/service/QaScenarioRetryService.java) (내부 호출 전용):

```
1. Redis 잔여 정리:
     DEL qa:scenarios:{id}:scenarios
     DEL qa:scenarios:{id}:meta
     DEL qa:cancel:{id}
2. pipeline state 리셋:
     status = SCENARIO_GENERATING
     failureReason / detailStage / detailOrder / detailMessage / detailSource = null
3. QaAnalyzeService.analyze(reportId, briefS3Url, summaryS3Url) 재호출 → SQS publish
```

---

## 4. 변경된 / 새로 추가된 파일

| 파일 | 역할 |
|---|---|
| `infrastructure/redis/QaPipelineState.java` | `scenarioAttemptCount`, `lastValidationErrors` 필드 추가 |
| `domain/qa/validation/SuiteValidationResult.java` | 검증 결과 DTO (errors / warnings / valid) |
| `domain/qa/validation/ValidationRules.java` | step/matcher/signal/captureKind/capability 레지스트리 JSON 로더 |
| `domain/qa/validation/QaSuiteValidationService.java` | 검증 본체 (validator.js Java 포팅) |
| `domain/qa/service/QaScenarioRetryService.java` | 내부 호출 전용 retry (가드는 호출자가) |
| `domain/qa/service/QaPipelineOrchestrator.java` | `onScenarioGenerateDone` 에 검증 + retry 트리거 통합 |
| `resources/qa-validation/qa-step-contract.json` | Playwright config 미러 |
| `resources/qa-validation/qa-matcher-registry.json` | Playwright config 미러 |
| `resources/qa-validation/qa-signal-registry.json` | Playwright config 미러 |
| `resources/qa-validation/qa-capture-kinds.json` | Playwright config 미러 |
| `resources/qa-validation/qa-capability-registry.json` | registry.js 의 CAPABILITY_REGISTRY 미러 |

---

## 5. 프론트 입장

✅ **API 변경 없음** — `/api/qa/start` 호출 후 WebSocket 으로 흐름 관찰만 하면 됨.

✅ **추가 처리 없음** — retry 가 일어나도 프론트는 `SCENARIO_GENERATE_STARTED → ...` 이벤트가 다시 들어오는 것으로만 인지. 기존 핸들러 그대로.

⚠️ **타이밍 변화** — 검증 통과까지 `READY_TO_EXECUTE` 가 지연됨 (최악 3회 = 3 × AI 추론 시간). 프론트의 "시나리오 생성 중" 표시 시간이 늘어날 수 있음. 디테일 단계는 `SCENARIO_GENERATING` 으로 유지되므로 별도 UI 변경 불필요.

---

## 6. SQS race 분석 (큐 잔존 메시지)

자세한 내용은 [worker.py](../ai/app/worker.py) 의 실패 처리 정책 참조. 요약:

| AI 실패 종류 | DeleteMessage | 큐 잔존 |
|---|---|---|
| INVALID_INPUT / S3_FETCH_FAILED (영구 실패) | ✅ | 사라짐 |
| MODEL_TIMEOUT / OOM / REDIS_SAVE_FAILED / UNKNOWN (일시 실패) | ❌ | 600s 후 SQS 자동 재배달 |

내부 retry 후 큐에 옛 메시지 + 새 메시지 2건이 있어도:
- AI Worker 가 단일 스레드 직렬 처리 ([worker.py:371](../ai/app/worker.py#L371) `MaxNumberOfMessages=1`)
- 멱등성 체크 ([worker.py:225-236](../ai/app/worker.py#L225-L236)) — 첫 처리 후 `meta` 키 생기면 두 번째 메시지는 자동 스킵
- → **한 번만 처리됨, race 안전**

retry 시 `:meta` 키를 지우는 동안 옛 메시지가 들어오면 옛 입력으로 처리될 수도 있지만, 같은 reportId · 같은 입력이라 결과 동일 (멱등).

---

## 7. 한계 & 향후

- **재시도 횟수 한도 = 3 회 (하드코딩)** — `QaPipelineOrchestrator.MAX_SCENARIO_ATTEMPTS`. 운영 중 늘릴 일 있으면 설정값으로 빼야.
- **검증 규칙 동기화** — playwright/config 와 backend/resources/qa-validation 둘 다 같은 JSON 유지 필요. 한쪽만 바꾸면 두 검증기가 어긋남.
- **분석 단계 retry 는 별도** — AI 가 분석 자체를 부실하게 끝낸 경우 (elements 비어있음 등) 는 시나리오 retry 로 해결 불가. 처음부터 재실행 필요.
- **timeout 검증** — `timeoutMs > MAX` 와 `<= 0` 모두 error 처리. timeoutMs 가 number 가 아닌 경우 자동 reject.

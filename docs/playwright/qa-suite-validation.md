# QA Suite 유효성 검사 명세

## 개요

AI가 생성한 시나리오 JSON은 Redis에 임시 저장된 뒤 `QaExecuteService`에 의해 S3에 조립·업로드되고, Playwright Worker로 전달된다.  
**실행 전 Spring에서 시나리오 JSON이 실행 가능한지 검증**해야 한다. 검증 실패 시 Worker에 SQS 메시지를 보내지 않고, 실패 이유를 반환하여 AI가 시나리오를 재생성할 수 있도록 한다.

```
S3 final-report.json (분석 결과) ──┐
                                   ├──→ [Spring 유효성 검사] → S3 suite.json → SQS → Playwright Worker
Redis (AI 시나리오 meta+scenarios) ─┘         ↓ 실패
                                         에러 이유 반환 → AI 재생성
```

검증은 **반드시 두 데이터를 모두 사용**한다:
- `final-report.json` (S3) — 분석에서 추출된 elements, nodeId 목록
- AI 생성 시나리오 JSON (Redis `qa:scenarios:{reportId}:meta` + `qa:scenarios:{reportId}:scenarios`)

---

## 검증 데이터 흐름

| 단계 | 위치 | 데이터 |
|---|---|---|
| AI 생성 결과 저장 | Redis `qa:scenarios:{reportId}:meta` | suite 메타 (suiteId, environment 등) |
| AI 생성 결과 저장 | Redis `qa:scenarios:{reportId}:scenarios` | 시나리오 배열 JSON |
| 분석 결과 | Redis `QaPipelineState.analysisS3Url` | S3 경로 |
| 분석 리포트 본문 | S3 `analysisS3Url` | elements 배열 포함 (nodeId 검증용) |
| 검증 통과 후 | S3 `jobs/{reportId}/result/scenarios.json` | suite.json (assembled) |

---

## Suite 최상위 구조 검증

Redis에서 읽은 meta + scenarios를 합친 suite JSON에 대해 검증.

| 필드 | 타입 | 필수 | 검증 규칙 |
|---|---|---|---|
| `suiteId` | string | **필수** | 비어있으면 오류 |
| `scenarios` | array | **필수** | 배열이어야 함, 비어있으면 오류 |
| `environment.baseURL` | string | 권장 | 없으면 경고 (goto 상대경로 실패 가능) |
| `defaults.executionPolicy.maxStepRetries` | number | 선택 | number 타입이어야 함 |
| `defaults.executionPolicy.timeoutMs` | number | 선택 | 양수, 최대 120000ms |
| `flows` | object | 선택 | 반복 step 묶음. `flowRef` 전처리에서 참조 |
| `dataSets` | object | 선택 | 데이터드리븐 실행 row 묶음. `dataSetRef` 전처리에서 참조 |

---

## Scenario 구조 검증

`scenarios` 배열 내 각 항목에 대해 검증.

| 필드 | 타입 | 필수 | 검증 규칙 |
|---|---|---|---|
| `scenarioId` | string | **필수** | 비어있으면 오류 |
| `title` | string | **필수** | 비어있으면 오류 |
| `steps` | array | 권장 | 없거나 비어있으면 경고 |
| `preconditions` | array | 선택 | steps와 합쳐서 검증 |
| `data` | object | 선택 | `${data.xxx}` 템플릿 참조값 |
| `dataSetRef` | string | 선택 | `suite.dataSets[dataSetRef]` row별로 시나리오를 확장 |
| `dataRows` | array | 선택 | scenario 내부 inline 데이터 row. `dataSetRef`보다 우선 |
| `flowRef` | string | 선택 | scenario 시작부에 삽입할 단일 reusable flow |
| `flowRefs` | array | 선택 | scenario 시작부에 순서대로 삽입할 reusable flow 목록 |
| `flowData` | object | 선택 | scenario-level flow parameter. `${flow.xxx}` 전처리에 사용 |

---

## 실행 전처리 확장

Playwright Worker는 실행과 `/qa/validate` HTTP 검증 전에 suite를 한 번 전처리한다. 전처리 결과는 기존 runner가 이해하는 순수 `scenarios[].steps[]` 형태가 되므로, 기존 step executor와 validator를 그대로 재사용한다.

처리 순서:

1. `flowRef` / `flowRefs` / `type: "flow"` step을 `suite.flows`의 실제 step 배열로 확장
2. `dataSetRef` / `dataRows`를 row별 scenario로 확장
3. 확장된 suite에 대해 기존 step validation과 실행 수행

### `flows` 형식

`suite.flows[flowId]`는 step 배열이거나 `{ "steps": [...] }` 형식이다. flow 내부 문자열에서 `${flow.xxx}`는 flow 호출부의 `data` 또는 `params` 값으로 전처리된다. `${data.xxx}`, `${captured.xxx}` 등 runtime 변수는 전처리 후에도 그대로 남아 실행 시 `RuntimeState`가 해석한다.

```json
{
  "flows": {
    "search.basic": [
      { "stepId": "fill", "type": "fill", "targetRef": { "nodeId": "search_input" }, "input": { "valueTemplate": "${flow.keyword}" } },
      { "stepId": "submit", "type": "press", "key": "Enter" }
    ]
  },
  "scenarios": [
    {
      "scenarioId": "SEARCH",
      "title": "검색",
      "steps": [
        { "stepId": "search", "type": "flow", "flowRef": "search.basic", "data": { "keyword": "${data.keyword}" } }
      ]
    }
  ]
}
```

### `dataSets` 형식

`suite.dataSets[dataSetId]`는 row 배열이거나 `{ "rows": [...] }`, `{ "data": [...] }`, `{ "values": [...] }` 형식이다. 각 row는 `{ "id", "label", "data" }`를 권장하며, `data`가 없으면 `id`, `label`, `name`, `title`, `description`, `tags`를 제외한 필드를 scenario data로 사용한다.

```json
{
  "dataSets": {
    "searchKeywords": [
      { "id": "korean", "label": "한글 검색어", "data": { "keyword": "자동화" } },
      { "id": "english", "label": "영문 검색어", "data": { "keyword": "playwright" } }
    ]
  },
  "scenarios": [
    {
      "scenarioId": "SEARCH",
      "title": "검색 결과 확인",
      "dataSetRef": "searchKeywords",
      "steps": [
        { "stepId": "search", "type": "flow", "flowRef": "search.basic", "data": { "keyword": "${data.keyword}" } }
      ]
    }
  ]
}
```

위 예시는 실행 전에 `SEARCH__korean`, `SEARCH__english` 두 시나리오로 확장된다. 확장된 scenario report에는 아래 메타데이터가 남는다.

| 필드 | 의미 |
|---|---|
| `dataDriven.sourceScenarioId` | 확장 전 원본 scenarioId |
| `dataDriven.dataSetRef` | 참조한 dataSet ID |
| `dataDriven.rowId` | row 식별자 |
| `dataDriven.rowIndex` | row 순서 |
| `dataDriven.rowLabel` | report 표시용 label |

---

## Step 구조 검증

`preconditions` + `steps`를 합쳐서 순서대로 검증.

### 공통 필드

| 필드 | 타입 | 검증 규칙 |
|---|---|---|
| `stepId` | string | 없으면 경고 |
| `type` | string | **필수**, 지원 타입 목록에 있어야 함 |
| `timeoutMs` | number | 양수, 최대 120000ms |

### 지원 step type 및 필수 필드

| type | 필수 필드 | targetRef 필요 | 비고 |
|---|---|---|---|
| `goto` | `url` | 불필요 | url은 `/` 또는 `http` 시작이어야 함 (경고) |
| `fill` | `targetRef`, `input` | **필요** | |
| `click` | `targetRef` | **필요** | container 클릭 시 경고, form 클릭 시 오류 |
| `expect` | `assertion` | 선택적 | assertion 내부 별도 검증 |
| `capture` | `targetRef` | 페이지 scope 제외 | capture.kind 별도 검증 |
| `scroll` | `scroll` | 불필요 | scroll.direction, scroll.pixels 필수 |
| `scrollToElement` | `targetRef` | **필요** | |
| `waitFor` | 없음 | 선택적 | waitFor.kind 가 timeout이면 waitFor.ms 필수 (최대 60000ms) |
| `select` | `targetRef`, `input` | **필요** | |
| `check` | `targetRef` | **필요** | |
| `uncheck` | `targetRef` | **필요** | |
| `press` | `key` | 선택적 | |
| `action` | `actionRef` | 선택적 | actionRef.capability 별도 검증 |
| `flow` | `flowRef` | 불필요 | 실행 전 `suite.flows`의 concrete step으로 확장 |

---

## Assertion (expect step) 검증

| 필드 | 규칙 |
|---|---|
| `assertion.matcher` | **필수**, 지원 matcher 목록에 있어야 함 |

### 지원 matcher 목록

| matcher | scope | 필수 필드 |
|---|---|---|
| `toHaveURL` | page | `value` |
| `toContainURL` | page | `value` |
| `toBeVisible` | element | - |
| `toBeHidden` | element | - |
| `toHaveText` | element | `value` |
| `toContainText` | element | `value` |
| `toHaveValue` | element | `value` |
| `toHaveAttribute` | element | `attribute`, `value` |
| `toHaveCountGreaterThan` | element | `value` |
| `toChangeFromStored` | element | `storedKey` |
| `textOrAriaStateChanged` | element | - |
| `toHaveScrollYLessThanOrEqual` | page | `value` |
| `toSatisfyAny` | page | `value` |
| `toHaveNoConsoleErrors` | page | - |
| `toHaveNoNetworkErrors` | page | - |

**추가 규칙:**
- `toHaveURL` / `toContainURL`을 popup 시그널 직후 step에서 사용하면 오류 (popupUrlContains 사용 권장)
- `toChangeFromStored`의 `storedKey`가 이전 capture step에서 아직 저장되지 않은 시점이면 경고

---

## Capture Step 검증

| capture.kind | 필수 필드 |
|---|---|
| `text` | - |
| `innerText` | - |
| `textContent` | - |
| `value` | - |
| `attribute` | `capture.attributeName` |
| `aria` | - |
| `screenshot` | - |
| `url` | - |
| `scrollY` | - |
| `visible` | - |
| `localStorage` | `capture.storageKey` |
| `sessionStorage` | `capture.storageKey` |
| `cookie` | `capture.cookieName` |

**추가 규칙:**
- `capture.saveAs` (또는 step 최상위 `saveAs`) 없으면 경고 (캡처 결과가 버려짐)
- 지원 목록에 없는 kind → 오류

---

## Signal (expectedSignals) 검증

| signal.type | required | 필수 필드 |
|---|---|---|
| `urlChanged` | true | - |
| `urlChangedOptional` | false | - |
| `networkRequest` | false | `urlContains` |
| `popupOpened` | true | - |
| `popupUrlContains` | true | `urlContains` |
| `domChanged` | true | - |
| `domChangedOptional` | false | - |
| `scrollChanged` | false | - |
| `elementVisible` | false | `targetRef` |

**추가 규칙:**
- `signal.type` 없으면 오류
- 지원 목록에 없는 type → 오류
- step type별 허용 signal: qa-step-contract.json의 `allowedSignals` 참조

---

## Action Step (actionRef) 검증

| actionRef.capability | 필수 필드 | targetRef 권장 |
|---|---|---|
| `common.openExternalLink` | - | 권장 |
| `common.submitSearch` | `query` | 권장 |
| `helpDocument.open` | - | 권장 |
| `institution.selectRelatedSite` | `value` | 권장 |

**추가 규칙:**
- `actionRef.capability` 없으면 오류
- 지원 목록에 없는 capability → 오류

---

## Analysis nodeId 일관성 검증 (**필수** — final-report.json 기반)

`QaPipelineState.analysisS3Url`에서 `final-report.json`을 로드하여 교차 검증한다.  
분석 리포트 없이 실행 요청이 오면 검증 자체를 오류로 처리해야 한다.

| 검증 항목 | 규칙 |
|---|---|
| `targetRef.nodeId` 존재 여부 | 분석 리포트 elements에 없는 nodeId → 오류 |
| 페이지 경로 일치 | nodeId가 속한 페이지 URL이 현재 활성 페이지(마지막 goto 경로)와 달라야 함 → 오류 |
| 네비게이션 후 nodeId 재사용 | urlChanged/urlChangedOptional 시그널 이후, 새 goto 없이 이전 페이지 nodeId 사용 → 오류 |
| 클릭 타겟 actionability | form 태그를 click으로 → 오류 / container 태그를 click으로 → 경고 |

---

## 변수 참조(`${}`) 검증

| 참조 패턴 | 검증 규칙 |
|---|---|
| `${data.xxx}` | scenario.data에 `xxx` 키 존재 여부 확인 |
| `${captured.xxx}` | 이전 capture step의 saveAs에 `xxx`가 있어야 함 (없으면 경고) |
| `${credential.xxx}` | 형식만 확인 (런타임 값) |
| `${runtime.xxx}` | 형식만 확인 (런타임 값) |
| `${flow.xxx}` | flow 전처리 전용. `type: "flow"` 호출부의 `data`/`params`에서 해석 |
| namespace 없는 `${xxx}` | scenario.data에 선언 없으면 경고 |

---

## 오류 vs 경고 분류

| 구분 | 의미 | 처리 방침 |
|---|---|---|
| **오류(error)** | 실행 불가 → Worker에 전달하면 반드시 실패함 | 검증 실패 반환, SQS 전송 차단, AI 재생성 유도 |
| **경고(warning)** | 실행은 되지만 의도와 다를 수 있음 | 검증 통과, 경고 내용을 응답에 포함 |

---

## Spring 구현 방향

### 검증 API 엔드포인트 (예시)

```
POST /api/qa/validate/{reportId}
Response:
{
  "valid": false,
  "errors": [
    "[scenario SC-001] step[s2] targetRef.nodeId \"btn_login\" does not exist in analysis report",
    "[scenario SC-001] step[s3] Unsupported matcher: \"toBeFocused\""
  ],
  "warnings": [
    "[scenario SC-002] capture step has no saveAs; value will be discarded"
  ]
}
```

### 검증 시점

```
QaExecuteService.execute()
  → suiteAssembleService.assembleAndUpload()  // Redis → S3
  → [신규] suiteValidationService.validate()  // S3 suite + analysisReport 로드 후 검증
      ├── valid=true  → sqsPublisher.publishExecute()
      └── valid=false → 오류 반환 (SQS 전송 안 함)
```

### 필요한 데이터 로드 (두 소스 모두 필수)

| 소스 | 경로 | 용도 |
|---|---|---|
| S3 `final-report.json` | `QaPipelineState.analysisS3Url` | elements 배열 → nodeId 존재·페이지 일치 검증 |
| Redis AI 시나리오 | `qa:scenarios:{reportId}:meta` + `qa:scenarios:{reportId}:scenarios` | 시나리오 구조 검증 |

`analysisS3Url`이 Redis에 없거나 S3 다운로드 실패 시 → 검증 오류 반환 (실행 불가 상태)

### 참조 소스

Playwright Worker의 `src/core/qa/validator.js`에 동일한 검증 로직이 구현되어 있다.  
`flowRef`와 `dataSetRef`는 `src/core/qa-execution/suitePreprocessor.js`에서 먼저 확장된 뒤 검증·실행된다.

Spring 구현 시에는 아래 둘 중 하나를 선택한다.

- Java 검증기에 `suitePreprocessor.js`와 같은 전처리 규칙을 포팅한 뒤 확장된 suite를 검증한다.
- Playwright Worker의 `/qa/validate` 엔드포인트를 호출해 전처리와 검증을 위임한다.

전처리 없이 `flow` step만 구조적으로 허용하면 `suite.flows` 누락이나 data row 변수 누락을 SQS 발행 전에 잡지 못할 수 있으므로, 운영 검증에서는 전처리 후 validator를 기준으로 삼는다.

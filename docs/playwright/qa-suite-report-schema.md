# QA Suite Report JSON 스키마

> **대상**: Playwright Worker가 생성하는 `suite-report.json` 포맷 명세.  
> Backend가 S3에서 읽어 프론트에 전달하거나, `qa-dashboard.html`이 직접 로드할 때 이 구조를 기준으로 한다.

---

## 최상위 구조

```jsonc
{
  "suiteName":  "www.example.site 전체 QA",   // 실행 대상 사이트명
  "baseURL":    "https://www.example.site",    // 분석 기준 URL
  "status":     "passed | partial | failed",  // 전체 suite 상태
  "startedAt":  "2026-05-13T09:12:00.000Z",   // ISO 8601
  "finishedAt": "2026-05-13T09:18:41.000Z",
  "durationMs": 401000,                        // 총 실행 시간 (ms)
  "summary":   { ... },   // ▶ 아래 참조
  "metrics":   { ... },   // ▶ 아래 참조 (없으면 클라이언트가 buildClientMetrics로 추론)
  "scenarioReports": [ ... ]  // ▶ 아래 참조
}
```

### `status` 값

| 값 | 조건 |
|----|------|
| `passed`  | 모든 시나리오 `passed` |
| `partial` | 일부 `partial` 또는 `failed`, 핵심 기능 통과 |
| `failed`  | 1개 이상 `failed`, 또는 critical 시나리오 실패 |

---

## `summary`

시나리오 단위 집계.

```jsonc
{
  "totalScenarios": 8,
  "passed":  5,
  "partial": 1,
  "failed":  2,
  "skipped": 0,
  "blocked": 0
}
```

---

## `metrics`

품질 점수 및 커버리지 분석. **선택 필드** — 없으면 대시보드가 `scenarioReports`에서 클라이언트 측 추론값을 사용한다.

```jsonc
{
  "healthScore":        72,     // 0–100. passRate·에러 분포 가중 합산
  "passRate":           75,     // (passed + partial) / total * 100
  "flakyRate":          8,      // 재시도 후 통과 비율 (%)
  "assertionCoverage":  62,     // expect 스텝 보유 시나리오 비율 (%)
  "locatorStability":   81,     // 폴백 없이 직접 해소된 로케이터 비율 (%)
  "selfHealingRate":    4,      // 셀프힐링 적용 스텝 비율 (%)
  "scenarioDepth":      7,      // 시나리오 평균 step 수
  "criticalPathPassed": true,   // `critical` 태그 시나리오 전체 통과 여부 (null = 태그 없음)

  // 기능 태그별 통과율 (실행 계층 @smoke 등 제외, 도메인 태그만)
  "featureCoverage": [
    { "tag": "auth",    "passed": 2, "total": 2, "passRate": 100 },
    { "tag": "cart",    "passed": 1, "total": 2, "passRate": 50  }
  ],

  // 실행 계층 태그(@smoke, @regression 등)별 통과율
  "tierCoverage": [
    { "tag": "@smoke", "label": "Smoke", "passed": 3, "total": 3, "passRate": 100 }
  ],

  // 에러 코드 분포 (상위 최대 10개)
  "errorDistribution": [
    { "code": "target_not_found",   "count": 5 },
    { "code": "timeout",            "count": 3 },
    { "code": "assertion_failed",   "count": 2 },
    { "code": "navigation_blocked", "count": 1 }
  ]
}
```

> `featureCoverage`와 `tierCoverage`가 모두 없으면 대시보드가 `tags` 배열에서 자동 분류한다.

---

## `scenarioReports[]`

시나리오별 실행 결과 배열.

```jsonc
{
  "scenarioId":   "VELOG-E2E-059",
  "scenarioName": "태그 필터 페이지 이동 확인",
  "status": "passed | partial | failed | skipped | blocked",
  "durationMs": 12400,
  "tags": ["search", "critical"],

  "summary": { ... },         // ▶ scenarioSummary 참조
  "humanNotes": [ "..." ],    // 사람이 읽을 수 있는 실행 요약 메시지 목록
  "stepResults": [ ... ]      // ▶ stepResult 참조
}
```

### `scenarioSummary`

```jsonc
{
  "total":   6,
  "passed":  6,
  "failed":  0,
  "skipped": 0,
  "retried": 0,
  "assertionPassedCount":       2,
  "assertionFailedCount":       0,
  "fallbackLocatorUsageCount":  0,
  "selfHealingCount":           0,
  "errorClassification": {      // failed 스텝에서 집계된 에러 코드 카운트
    "target_not_found": 1
  }
}
```

---

## `stepResults[]`

단일 스텝 실행 결과.

```jsonc
{
  "stepId":     "step-02",              // 시나리오 내 고유 식별자
  "type":       "goto | fill | click | expect | check | select | hover | ...",
  "status":     "passed | failed | skipped | blocked | retried_then_passed",
  "durationMs": 420,

  // 실패 시에만 존재
  "error":     "target_not_found: 수량 감소 버튼을 찾을 수 없습니다.",
  "errorCode": "target_not_found",

  // 로케이터 복구가 사용된 경우에만 존재
  "resolutionResult": {
    "method":          "selfHealing | locatorFallback",
    "wasFallbackUsed": true,
    "healingScore":    7        // selfHealing 전용
  },

  // 스크린샷 아티팩트 (captureOnSuccess=true 또는 실패 시)
  "artifacts": [
    { "fileName": "step-02-passed-s2.png" }
  ]
}
```

### `status` 값

| 값 | 의미 |
|----|------|
| `passed`               | 정상 통과 |
| `failed`               | 실패 (이후 스텝 blocked 처리) |
| `skipped`              | 선행 실패로 건너뜀 |
| `blocked`              | 실행 자체 불가 |
| `retried_then_passed`  | 재시도 후 통과 (partial 원인) |

### 에러 코드 목록

| 코드 | 설명 |
|------|------|
| `target_not_found`   | 로케이터가 페이지에 존재하지 않음 |
| `timeout`            | 지정 시간 내 요소 미출현 또는 응답 없음 |
| `assertion_failed`   | expect 조건 불일치 |
| `navigation_blocked` | 페이지 이동 차단 (리다이렉트, CSP 등) |
| `locator_ambiguous`  | 동일 조건 요소 복수 매칭 |
| `step_error`         | 기타 실행 오류 |

---

## 스크린샷 아티팩트 파일명 규칙

```
{phase}-{index:02d}-{status}-{stepKey}.png
```

| 세그먼트 | 예시 | 설명 |
|----------|------|------|
| `phase`  | `step`, `pre`, `post` | 실행 단계 |
| `index`  | `01`, `02` | 0-패딩 스텝 번호 |
| `status` | `passed`, `failed`, `skipped` | 스텝 상태 |
| `stepKey`| `s2`, `pre-1` | 시나리오 내 stepId 짧은 키 |

예: `step-02-passed-s2.png`, `pre-01-passed-pre-1.png`

---

## 대시보드 (`qa-dashboard.html`) 사용법

### 파일 업로드 / 직접 입력

`playwright/public/qa-dashboard.html`을 브라우저에서 열고 JSON 파일을 선택하거나 텍스트 영역에 붙여넣은 후 **대시보드 적용** 버튼을 클릭한다.

### URL 쿼리 파라미터로 자동 로드

```
qa-dashboard.html?report=https://cdn.example.com/reports/suite-report.json
```

`report` 파라미터로 JSON URL을 전달하면 페이지 로드 시 자동으로 fetch 후 렌더링한다.

### `metrics` 없는 JSON 처리

`metrics` 필드가 없거나 빈 객체인 경우 대시보드가 `scenarioReports`에서 다음 값을 클라이언트 측 추론한다:

| 추론 항목 | 방법 |
|-----------|------|
| `passRate` | `(passed + partial) / total * 100` |
| `healthScore` | `passRate * 0.55 + errorPenalty * 0.45` |
| `featureCoverage` | `tags` 배열 집계 |
| `errorDistribution` | `summary.errorClassification` 합산 (상위 5개) |
| 나머지 (flakyRate 등) | `null` — 대시보드에서 `—` 표시 |

---

## 관련 문서

- [qa-suite-validation.md](./qa-suite-validation.md) — AI 생성 시나리오 유효성 검사 규칙
- [qa-pipeline-sqs-contract.md](./qa-pipeline-sqs-contract.md) — SQS 요청/응답 계약
- [qa-ai-scenario-generation-contract.md](./qa-ai-scenario-generation-contract.md) — AI 시나리오 생성 입력 계약
- [../backend/qa-pipeline-state-api.md](../backend/qa-pipeline-state-api.md) — 파이프라인 상태 조회 API

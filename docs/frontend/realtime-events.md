# Playwright 실시간 이벤트 명세

이 문서는 Frontend가 분석 화면과 QA 실행 화면을 실시간으로 갱신하기 위해 받는 Redis/WebSocket 이벤트 규격이다.

핵심 원칙은 단순하다.

- Redis/WebSocket에는 화면에 바로 필요한 최소 요약만 보낸다.
- 사람이 보는 타임라인 문구는 `message` 필드에 한국어로 보낸다.
- 원본 DOM, 전체 리포트, 전체 시나리오 결과, 이미지/영상 바이너리는 보내지 않는다.
- 전체 데이터가 필요하면 이벤트에 포함된 분석 리포트 S3 링크를 다시 조회한다.
- 분석 그래프는 Frontend가 페이지 단위 이벤트의 `analysis.page.discoveredFrom`을 누적해 구성한다.
- `log`는 기존 호환용으로만 허용한다. 새 이벤트에서는 `message`와 중복이면 생략한다.

## 전송 경로

```text
Playwright Worker
  -> Redis Pub/Sub autoqa:progress:{jobId}
  -> Backend WebSocket
  -> Frontend
```

Redis 키:

```text
autoqa:progress:{jobId}      # 실시간 Pub/Sub 채널
autoqa:progress:log:{jobId}  # 최근 이벤트 로그 리스트
autoqa:status:{jobId}        # 최신 상태 스냅샷
```

기존 `kind: "log"`, `kind: "status"` 메시지는 유지된다. 새 화면은 `kind: "realtime-event"`, `eventVersion: "2.4"` 메시지를 사용한다.

## 공통 Envelope

모든 구조화 이벤트는 아래 공통 필드를 가진다.

```json
{
  "eventVersion": "2.4",
  "producer": "playwright-worker",
  "kind": "realtime-event",
  "eventType": "ANALYSIS_PAGE_COMPLETED",
  "timestamp": "2026-05-06T05:00:00.000Z",
  "sequence": 7,
  "jobId": "42",
  "projectId": 1,
  "jobType": "PAGE_ANALYSIS",
  "testType": "crawl",
  "stage": "analysis",
  "status": "RUNNING",
  "message": "페이지 3 분석 완료: https://example.com/products (Products)."
}
```

| 필드 | 설명 |
|---|---|
| `eventVersion` | 구조화 이벤트 버전. 현재 `2.4`. |
| `kind` | 구조화 이벤트는 항상 `realtime-event`. |
| `eventType` | 이벤트 종류. 아래 목록 참고. |
| `timestamp` | 이벤트 발생 시각. |
| `sequence` | job 내 이벤트 순서. 화면 정렬에 사용한다. |
| `jobId` | 작업 ID. |
| `stage` | `analysis` 또는 `execute`. |
| `status` | 작업/이벤트 상태. `RUNNING`, `PASSED`, `PARTIAL`, `FAILED`, `ERROR`, `SKIPPED`, `BLOCKED`. |
| `message` | 사용자에게 보여줄 한국어 문구. 타임라인 기본 텍스트로 사용한다. |
| `log` | 레거시 호환 필드. `message`와 같으면 생략된다. |

진행률은 필요한 이벤트에만 포함된다.

```json
{
  "progress": {
    "scope": "analysis",
    "unit": "page",
    "current": 3,
    "total": 10,
    "percent": 30
  }
}
```

## 분석 화면 이벤트

사용자가 URL로 분석을 시작하면 페이지 분석이 하나씩 완료될 때마다 이벤트가 온다. Frontend는 이 이벤트를 받아 URL 그래프와 페이지 요약 카드를 갱신한다.

### `ANALYSIS_STARTED`

분석 시작 이벤트다.

```json
{
  "eventType": "ANALYSIS_STARTED",
  "stage": "analysis",
  "status": "RUNNING",
  "message": "URL 분석을 시작합니다: https://example.com",
  "progress": { "scope": "analysis", "unit": "page", "current": 0, "total": 20, "percent": 0 },
  "target": { "url": "https://example.com" }
}
```

### `ANALYSIS_PAGE_COMPLETED`

페이지 하나의 분석이 끝날 때마다 발행된다. 전체 사이트 그래프나 연결 목록은 보내지 않고, 현재 페이지가 어떤 페이지에서 발견되었는지에 대한 최소 연결 정보만 보낸다. Frontend는 각 페이지 이벤트를 누적해 URL 탐색 그래프를 구성한다.

```json
{
  "eventType": "ANALYSIS_PAGE_COMPLETED",
  "stage": "analysis",
  "status": "RUNNING",
  "message": "페이지 3 분석 완료: https://example.com/products (Products).",
  "progress": { "scope": "analysis", "unit": "page", "current": 3, "total": 10, "percent": 30 },
  "analysis": {
    "page": {
      "pageIndex": 3,
      "url": "https://example.com/products",
      "path": "/products",
      "title": "Products",
      "status": "analyzed_successfully",
      "discoveredFrom": {
        "pageIndex": 1,
        "url": "https://example.com",
        "path": "/"
      }
    }
  }
}
```

페이지 분석 실패 또는 확인 필요 상태도 같은 이벤트로 온다.

```json
{
  "eventType": "ANALYSIS_PAGE_COMPLETED",
  "stage": "analysis",
  "status": "RUNNING",
  "message": "페이지 4 분석 확인 필요: https://example.com/login. 이유: 자동화 방지 화면이 감지되었습니다.",
  "progress": { "scope": "analysis", "unit": "page", "current": 4, "total": 10, "percent": 40 },
  "analysis": {
    "page": {
      "pageIndex": 4,
      "url": "https://example.com/login",
      "path": "/login",
      "status": "manual_verification_required",
      "failureReason": "자동화 방지 화면이 감지되었습니다.",
      "discoveredFrom": {
        "pageIndex": 3,
        "url": "https://example.com/products",
        "path": "/products"
      }
    }
  }
}
```

Frontend 처리:

- `analysis.page.pageIndex`를 node id로 사용해 페이지 카드와 그래프 node를 upsert한다.
- `analysis.page.discoveredFrom.pageIndex`가 있으면 `discoveredFrom.pageIndex -> pageIndex` edge를 만든다.
- 그래프 카운트는 Frontend의 누적 `nodes`/`edges` 상태에서 계산한다.
- `analysis.page.failureReason`이 있으면 실패/확인 필요 이유로 표시한다.
- 타임라인에는 `message`를 그대로 표시한다.

### `ANALYSIS_ARTIFACTS_READY`

분석이 끝나고 S3 산출물 링크가 준비되면 발행된다. 전체 상세가 필요할 때만 S3 링크를 사용한다.

```json
{
  "eventType": "ANALYSIS_ARTIFACTS_READY",
  "stage": "analysis",
  "status": "PASSED",
  "message": "분석이 완료되었습니다. 10/10개 페이지를 처리했습니다.",
  "progress": { "scope": "analysis", "unit": "page", "current": 10, "total": 10, "percent": 100 },
  "analysis": {
    "summary": {
      "totalPages": 10,
      "completedPages": 10,
      "analyzedPages": 10,
      "failedPages": 0,
      "skippedPages": 0,
      "outOfScopePages": 0
    },
    "graph": {
      "nodeCount": 20,
      "edgeCount": 34
    },
    "failures": []
  },
  "artifacts": {
    "analysisS3Url": "s3://bucket/reports/42/analysis/final-report.json",
    "analysisSummaryS3Url": "s3://bucket/reports/42/analysis/analysis-summary.json"
  }
}
```

## QA 실행 화면 이벤트

QA 실행은 시나리오와 스텝 단위로 진행된다. Frontend는 진행 상태에 필요한 최소 id/index/status/reason/link만 받는다. selector 상세, console/network 상세, trace/video 링크, 전체 step 배열은 최종 리포트 S3에서 조회한다.

### `EXECUTION_REQUESTED`

QA 실행 입력이 준비된 상태다.

```json
{
  "eventType": "EXECUTION_REQUESTED",
  "stage": "execute",
  "status": "RUNNING",
  "message": "QA 실행 준비가 완료되었습니다. 시나리오 묶음: 로그인 스모크.",
  "execution": {
    "suiteId": "suite_login_smoke",
    "suiteName": "로그인 스모크"
  }
}
```

### `EXECUTION_SUITE_STARTED`

전체 QA 실행 시작 이벤트다.

```json
{
  "eventType": "EXECUTION_SUITE_STARTED",
  "stage": "execute",
  "status": "RUNNING",
  "message": "QA 실행을 시작합니다. 총 3개 시나리오를 실행합니다.",
  "progress": { "scope": "execute", "unit": "scenario", "current": 0, "total": 3, "percent": 0 },
  "execution": {
    "suiteId": "suite_login_smoke",
    "suiteName": "로그인 스모크",
    "scenarioCount": 3
  }
}
```

### `EXECUTION_SCENARIO_STARTED`

시나리오 하나가 시작될 때 온다.

```json
{
  "eventType": "EXECUTION_SCENARIO_STARTED",
  "stage": "execute",
  "status": "RUNNING",
  "message": "1/3번째 시나리오를 시작합니다: 로그인 성공",
  "progress": { "scope": "execute", "unit": "scenario", "current": 0, "total": 3, "percent": 0 },
  "scenario": {
    "scenarioId": "scenario_001",
    "scenarioName": "로그인 성공",
    "scenarioIndex": 1
  }
}
```

### `EXECUTION_STEP_STARTED`

스텝이 시작될 때 온다.

```json
{
  "eventType": "EXECUTION_STEP_STARTED",
  "stage": "execute",
  "status": "RUNNING",
  "message": "로그인 성공 - 1번 스텝 시작: 로그인 페이지 열기",
  "progress": { "scope": "scenario", "unit": "step", "current": 0, "total": 5, "percent": 0 },
  "scenario": {
    "scenarioId": "scenario_001",
    "scenarioIndex": 1
  },
  "step": {
    "stepIndex": 1,
    "stepId": "open_login",
    "name": "로그인 페이지 열기",
    "status": "running"
  }
}
```

### `EXECUTION_STEP_RETRY`

스텝 재시도 직전에 온다.

```json
{
  "eventType": "EXECUTION_STEP_RETRY",
  "stage": "execute",
  "status": "FAILED",
  "message": "로그인 성공 - 2번 스텝 재시도 1/2: 로그인 버튼 클릭",
  "progress": { "scope": "scenario", "unit": "step", "current": 2, "total": 5, "percent": 40 },
  "scenario": {
    "scenarioId": "scenario_001",
    "scenarioIndex": 1
  },
  "step": {
    "stepIndex": 2,
    "stepId": "click_login",
    "name": "로그인 버튼 클릭",
    "status": "failed",
    "retryCount": 1,
    "maxRetries": 2,
    "failure": {
      "reason": "요소를 찾을 수 없습니다.",
      "errorCode": "target_not_found",
      "screenshotAvailable": false
    }
  }
}
```

### `EXECUTION_STEP_FINISHED`

스텝이 끝날 때 온다. 실패한 경우 실패 이유와 실패 화면 생성 여부만 포함된다.

```json
{
  "eventType": "EXECUTION_STEP_FINISHED",
  "stage": "execute",
  "status": "FAILED",
  "message": "로그인 성공 - 2번 스텝 실패: 로그인 버튼 클릭. 이유: 요소를 찾을 수 없습니다. 실패 화면을 저장했습니다.",
  "progress": { "scope": "scenario", "unit": "step", "current": 2, "total": 5, "percent": 40 },
  "scenario": {
    "scenarioId": "scenario_001",
    "scenarioIndex": 1
  },
  "step": {
    "stepIndex": 2,
    "stepId": "click_login",
    "name": "로그인 버튼 클릭",
    "status": "failed",
    "durationMs": 10502,
    "retryCount": 2,
    "failure": {
      "reason": "요소를 찾을 수 없습니다.",
      "errorCode": "target_not_found",
      "screenshotAvailable": true
    }
  }
}
```

주의: `EXECUTION_STEP_FINISHED` 시점에는 실패 스크린샷이 로컬에 생성된 상태일 수 있다. Frontend에서 열 수 있는 S3 URL은 `EXECUTION_ARTIFACTS_READY` 이벤트의 `artifacts.failureScreenshots`에서 받는다.

### `EXECUTION_SCENARIO_FINISHED`

시나리오 하나가 끝났을 때 온다.

```json
{
  "eventType": "EXECUTION_SCENARIO_FINISHED",
  "stage": "execute",
  "status": "FAILED",
  "message": "1/3번째 시나리오 실패: 로그인 성공",
  "progress": { "scope": "execute", "unit": "scenario", "current": 1, "total": 3, "percent": 33 },
  "scenario": {
    "scenarioId": "scenario_001",
    "scenarioName": "로그인 성공",
    "scenarioIndex": 1,
    "status": "failed",
    "durationMs": 22112
  }
}
```

### `EXECUTION_ARTIFACTS_READY`

QA 실행이 끝나고 S3 링크가 준비되면 온다. 실패 스크린샷 S3 URL은 여기서 받는다.

```json
{
  "eventType": "EXECUTION_ARTIFACTS_READY",
  "stage": "execute",
  "status": "PARTIAL",
  "message": "QA 실행이 완료되었습니다. 통과 2개, 부분 성공 1개, 실패 0개입니다.",
  "progress": { "scope": "execute", "unit": "scenario", "current": 3, "total": 3, "percent": 100 },
  "execution": {
    "suiteId": "suite_login_smoke",
    "suiteName": "로그인 스모크",
    "summary": {
      "totalScenarios": 3,
      "passed": 2,
      "partial": 1,
      "failed": 0
    },
    "failedScenarios": [
      {
        "scenarioId": "scenario_001",
        "scenarioName": "로그인 성공",
        "status": "failed",
        "failedSteps": [
          {
            "stepIndex": 2,
            "stepId": "click_login",
            "name": "로그인 버튼 클릭",
            "failureReason": "요소를 찾을 수 없습니다.",
            "screenshotAvailable": true
          }
        ]
      }
    ]
  },
  "artifacts": {
    "resultS3Url": "s3://bucket/reports/42/execute/suite-report.json",
    "failureScreenshots": [
      {
        "scenarioId": "scenario_001",
        "stepId": "click_login",
        "s3Url": "s3://bucket/reports/42/execute/scenarios/scenario_001/screenshots/fail-01-click_login.png"
      }
    ]
  }
}
```

Frontend 처리:

- 시나리오 목록은 `EXECUTION_SCENARIO_STARTED`, `EXECUTION_SCENARIO_FINISHED`로 갱신한다.
- 스텝 상태는 `EXECUTION_STEP_STARTED`, `EXECUTION_STEP_RETRY`, `EXECUTION_STEP_FINISHED`로 갱신한다.
- 실패 이유는 `step.failure.reason` 또는 최종 이벤트의 `failedSteps[].failureReason`을 표시한다.
- 실패 스크린샷 링크는 `EXECUTION_ARTIFACTS_READY.artifacts.failureScreenshots`를 사용한다.
- selector, console/network 상세는 `EXECUTION_ARTIFACTS_READY.artifacts.resultS3Url`의 전체 리포트에서 조회한다.
- data-driven 원본 scenario/row 매핑, flow 전처리 통계, flake analytics, adaptive locator 상세도 전체 `suite-report.json`에서 조회한다.
- 타임라인에는 모든 이벤트의 `message`를 그대로 표시한다.

## 이벤트 종류 요약

| 이벤트 | 화면에서의 용도 |
|---|---|
| `PLAYWRIGHT_JOB_STARTED` | 전체 작업 시작 로그 |
| `PLAYWRIGHT_JOB_COMPLETED` | 전체 작업 완료 로그 |
| `PLAYWRIGHT_JOB_FAILED` | 전체 작업 실패 로그 |
| `ANALYSIS_STARTED` | URL 분석 시작 표시 |
| `ANALYSIS_PAGE_COMPLETED` | 페이지 카드와 URL 그래프 갱신 |
| `ANALYSIS_ARTIFACTS_READY` | 분석 최종 S3 링크 표시 |
| `EXECUTION_REQUESTED` | QA 실행 준비 표시 |
| `EXECUTION_SUITE_STARTED` | 시나리오 전체 진행률 초기화 |
| `EXECUTION_SCENARIO_STARTED` | 시나리오 시작 표시 |
| `EXECUTION_STEP_STARTED` | 스텝 진행 중 표시 |
| `EXECUTION_STEP_RETRY` | 스텝 재시도 표시 |
| `EXECUTION_STEP_FINISHED` | 스텝 결과/실패 이유 표시 |
| `EXECUTION_SCENARIO_FINISHED` | 시나리오 결과 표시 |
| `EXECUTION_ARTIFACTS_READY` | 실행 결과와 실패 스크린샷 S3 링크 표시 |

## 보내지 않는 데이터

아래 데이터는 Redis/WebSocket으로 보내지 않는다.

- 원본 DOM
- 전체 `final-report.json`
- 전체 `suite-report.json`
- 전체 step 배열
- screenshot/video/trace 바이너리
- trace.zip 링크
- raw network body

`trace.zip`은 현재 보존하지 않는다. Frontend는 trace 링크 UI를 숨긴다.

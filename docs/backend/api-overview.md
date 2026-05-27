# API 개요

Backend 서버(`http://localhost:8080`)가 제공하는 REST API와 WebSocket 엔드포인트 목록입니다.  
모든 REST 응답은 아래 공통 래퍼 형식을 따릅니다.

```json
{
  "success": true,
  "message": "ok",
  "data": { ... }
}
```

---

## Projects

### 프로젝트 목록 조회

```
GET /api/v1/projects
```

**응답 `data`**: `Project[]`

---

### 프로젝트 단건 조회

```
GET /api/v1/projects/{id}
```

---

### 프로젝트 생성

```
POST /api/v1/projects
```

**요청 바디**:
```json
{
  "name": "MyApp",
  "baseUrl": "https://example.com",
  "description": "선택 사항"
}
```

---

### 프로젝트 삭제

```
DELETE /api/v1/projects/{id}
```

---

### Project 스키마

```json
{
  "id": 1,
  "name": "MyApp",
  "baseUrl": "https://example.com",
  "description": null,
  "createdAt": "2026-04-20T09:00:00",
  "updatedAt": "2026-04-20T09:00:00"
}
```

---

## Test Jobs

### 잡 목록 조회 (프로젝트별)

```
GET /api/v1/jobs?projectId={projectId}
```

---

### 잡 단건 조회

```
GET /api/v1/jobs/{id}
```

---

### 잡 생성 (테스트 실행 요청)

```
POST /api/v1/jobs
```

**요청 바디**:
```json
{
  "projectId": 1,
  "testType": "smoke",
  "scenarioPrompt": "custom 타입일 때만 사용"
}
```

- `testType`: `smoke` | `regression` | `custom`
- `scenarioPrompt`: `custom` 타입일 때 AI에 전달할 시나리오 설명

생성 즉시 SQS 큐에 발행되며, Worker가 비동기로 처리합니다.

---

### 잡 상태 업데이트 (Worker → Backend)

```
PATCH /api/v1/jobs/{id}/status
```

**요청 바디**:
```json
{
  "status": "RUNNING",
  "reportPath": null
}
```

- `status`: `PENDING` | `RUNNING` | `PASSED` | `FAILED` | `ERROR`
- `reportPath`: 완료 시 MinIO 경로

---

### TestJob 스키마

```json
{
  "id": 42,
  "projectId": 1,
  "projectName": "MyApp",
  "status": "PASSED",
  "testType": "smoke",
  "scenarioPrompt": null,
  "reportPath": "http://localhost:9000/autoqa-results/reports/42/screenshot.png",
  "createdAt": "2026-04-20T09:00:00",
  "updatedAt": "2026-04-20T09:01:30"
}
```

---

## Test Results

### 잡 결과 조회

```
GET /api/v1/results/jobs/{jobId}
```

---

### 결과 저장 (Worker → Backend)

```
POST /api/v1/results
```

**요청 바디**:
```json
{
  "testJobId": 42,
  "totalTests": 3,
  "passedTests": 3,
  "failedTests": 0,
  "durationMs": 4200,
  "reportUrl": "http://localhost:9000/autoqa-results/reports/42/screenshot.png"
}
```

---

### TestResult 스키마

```json
{
  "id": 10,
  "testJobId": 42,
  "totalTests": 3,
  "passedTests": 3,
  "failedTests": 0,
  "durationMs": 4200,
  "reportUrl": "http://...",
  "createdAt": "2026-04-20T09:01:30"
}
```

---

## WebSocket

```
WS /ws/jobs/{jobId}/logs
```

연결 후 Worker가 Redis에 발행하는 실시간 진행 로그를 수신합니다.  
메시지 형식은 [realtime-events.md](realtime-events.md) 참조.

---

## AI 서버 (`http://localhost:8000`)

### 스크립트 생성

```
POST /api/generate
```

**요청 바디**:
```json
{
  "base_url": "https://example.com",
  "test_type": "custom",
  "scenario_prompt": "로그인 후 대시보드가 표시되는지 확인"
}
```

**응답**:
```json
{
  "script": "// AutoQA ...",
  "model_used": "rule-based-v1"
}
```

---

### 결과 분석

```
POST /api/analyze
```

**요청 바디**:
```json
{
  "job_id": 42,
  "total_tests": 3,
  "passed_tests": 2,
  "failed_tests": 1,
  "duration_ms": 5000,
  "error_messages": ["Timeout waiting for selector"]
}
```

**응답**:
```json
{
  "summary": "일부 테스트 실패 (1/3). 점검 필요.",
  "recommendations": ["실패한 테스트 케이스를 확인하세요.", "..."],
  "severity": "medium"
}
```

- `severity`: `low` | `medium` | `high`

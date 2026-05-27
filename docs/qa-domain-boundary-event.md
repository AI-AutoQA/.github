# 웹 분석 페이지 중단 이벤트

## 개요

웹 분석 크롤러가 페이지 탐색을 중단하는 이유는 여러 가지입니다.  
이 모든 경우는 **실패(failure)가 아닌 정보 이벤트**로 처리되며, 각각의 이유와 시각 정보가 프론트엔드에 전달됩니다.

---

## 중단 유형 전체 목록

| `status` | 의미 | `screenshotPresignedUrl` | `offendingUrl` | `failureReason` |
|----------|------|--------------------------|----------------|-----------------|
| `stopped_out_of_scope` | 요청 URL 자체가 rootHost 밖 (브라우저 미열림) | ❌ | ✅ 이탈 URL | ✅ 이유 문자열 |
| `stopped_redirect_out_of_scope` | 리다이렉트로 rootHost 이탈 | ✅ S3 설정 시 | ✅ 이탈 URL | ✅ 이유 문자열 |
| `manual_verification_required` | Captcha / 봇 감지 화면 | ✅ S3 설정 시 | ❌ | ✅ 감지된 챌린지 유형 |
| `failed` | 예외 발생 (네트워크 오류, 브라우저 크래시 등) | ❌ | ❌ | ✅ 오류 메시지 |
| `skipped_prior_run` | 이전 크롤 실행에서 이미 분석됨 | ❌ | ❌ | ❌ |

> robots.txt 차단, maxPages/maxDepth 초과로 큐에 아예 들어가지 않은 URL은 이벤트가 발생하지 않습니다.

---

## 이벤트 수신 경로

### 실시간 이벤트 (WebSocket / Redis Pub-Sub)

`ANALYSIS_PAGE_COMPLETED` 이벤트의 `analysis.page` 객체에 포함됩니다.

```json
{
  "eventType": "ANALYSIS_PAGE_COMPLETED",
  "stage": "analysis",
  "status": "RUNNING",
  "analysis": {
    "page": {
      "pageIndex": 3,
      "url": "https://example.com/checkout",
      "status": "stopped_redirect_out_of_scope",
      "failureReason": "최종 렌더링된 URL이 payment.external.com으로 이동하여 루트 호스트 example.com을 벗어났습니다",
      "offendingUrl": "https://payment.external.com/pay?session=abc123",
      "screenshotPresignedUrl": "https://s3.amazonaws.com/autoqa-artifacts-prod/...?X-Amz-Signature=...",
      "discoveredFrom": {
        "pageIndex": 1,
        "url": "https://example.com/cart"
      }
    }
  }
}
```

---

## 유형별 상세

### `stopped_out_of_scope`

링크에서 수집된 URL이 처음부터 다른 도메인을 가리키는 경우입니다.  
브라우저를 열기 전에 차단되므로 스크린샷이 없습니다.

```json
{
  "status": "stopped_out_of_scope",
  "failureReason": "요청 호스트 other-domain.com가 루트 호스트 example.com와 일치하지 않습니다",
  "offendingUrl": "https://other-domain.com/page",
  "screenshotPresignedUrl": null
}
```

---

### `stopped_redirect_out_of_scope`

페이지를 로드했더니 서버 리다이렉트로 다른 도메인으로 이동한 경우입니다.  
이탈 직전 화면 스크린샷이 S3에 저장되고 pre-signed URL이 제공됩니다.

```json
{
  "status": "stopped_redirect_out_of_scope",
  "failureReason": "최종 렌더링된 URL이 payment.external.com으로 이동하여 루트 호스트 example.com을 벗어났습니다",
  "offendingUrl": "https://payment.external.com/pay",
  "screenshotPresignedUrl": "https://s3.amazonaws.com/autoqa-artifacts-prod/.../baseline-annotated.png?X-Amz-Expires=3600&..."
}
```

---

### `manual_verification_required`

페이지에 Captcha / 봇 차단 화면이 감지된 경우입니다.  
`failureReason`에 감지된 챌린지 유형이 포함됩니다.

감지 유형:
- `captcha` — reCAPTCHA, hCaptcha, "로봇이 아닙니다" 체크박스
- `turnstile` — Cloudflare Turnstile
- `browser_verification` — "Checking your browser…" (Cloudflare 대기 화면)
- `automation_block` — "Access Denied", "Too many requests" 등

```json
{
  "status": "manual_verification_required",
  "failureReason": "자동화 차단 감지: captcha, browser_verification",
  "offendingUrl": null,
  "screenshotPresignedUrl": "https://s3.amazonaws.com/autoqa-artifacts-prod/.../baseline-annotated.png?X-Amz-Expires=3600&..."
}
```

---

### `failed`

`runAnalysis` 실행 중 예외가 발생한 경우입니다.  
`failureReason`에 오류 메시지가 그대로 포함됩니다.

```json
{
  "status": "failed",
  "failureReason": "net::ERR_CONNECTION_REFUSED at https://example.com/page",
  "offendingUrl": null,
  "screenshotPresignedUrl": null
}
```

---

## 프론트엔드 처리 가이드

### 1. 중단 유형별 분기 처리

```ts
switch (page.status) {
  case 'stopped_out_of_scope':
  case 'stopped_redirect_out_of_scope':
    // 도메인 이탈 — offendingUrl, screenshotPresignedUrl 활용
    break;
  case 'manual_verification_required':
    // 봇 감지 차단 — failureReason에 챌린지 유형, screenshotPresignedUrl 활용
    break;
  case 'failed':
    // 예외 — failureReason에 오류 메시지
    break;
  case 'skipped_prior_run':
    // 이전 분석 결과 재사용
    break;
}
```

### 2. 스크린샷 표시

`screenshotPresignedUrl`이 있으면 `<img>` 태그로 직접 사용 가능합니다.

```tsx
{page.screenshotPresignedUrl && (
  <img
    src={page.screenshotPresignedUrl}
    alt="중단 시점 스크린샷"
  />
)}
```

### 3. 중단 사유 표시

```ts
// failureReason: page.failureReason
// offendingUrl:  page.offendingUrl  (이탈 URL, 해당 시에만)
```

### 4. 실패로 처리하지 않기

`stopped_*`, `manual_verification_required`는 탐색 실패가 아닌 **정상 종료 사유**입니다.  
`failed`만 실제 오류로 분류하는 것을 권장합니다.

```ts
const stopReasons = ['stopped_out_of_scope', 'stopped_redirect_out_of_scope', 'manual_verification_required'];
const isStopEvent = stopReasons.includes(page.status);  // 중단 — 정보성
const isError     = page.status === 'failed';            // 오류 — 실패 카운트
```

---

## 주의사항

- `screenshotPresignedUrl`의 유효 시간은 **3600초(1시간)**입니다.
- S3가 설정되지 않은 환경(로컬 개발 등)에서는 `screenshotPresignedUrl`이 항상 `null`입니다.
- `stopped_out_of_scope`는 브라우저를 열기 전에 차단되므로 스크린샷이 없습니다.

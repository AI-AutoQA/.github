# AutoQA Playwright 엔진 발전 과정

AutoQA 서비스에서 Playwright 기반 웹 분석·실행 엔진이 어떤 문제를 만나고, 어떻게 진화해왔는지 단계별로 정리한다.  
각 단계는 `troubleshooting.md`의 실제 트러블슈팅 경험을 기반으로 한다.

---

## 요약

### 전체 발전 흐름

```
Phase 1 — 기초 안정화       : 크롤이 "돌아가게" 만들기
Phase 2 — 크롤 품질 향상    : 크롤이 "올바르게" 돌아가게 만들기
Phase 3 — 실시간·관측성    : 크롤 내부를 "볼 수 있게" 만들기
Phase 4 — QA 실행 안정성   : 시나리오가 "안정적으로" 실행되게 만들기
Phase 5 — 인증·인프라       : 운영 환경에서 "지속 가능하게" 만들기
```

### 한눈에 보기

| Phase | 해결한 핵심 문제 | 도입한 기술/패턴 | 영향 범위 |
|-------|-----------------|-----------------|-----------|
| 1 | SPA 중간 네비게이션으로 V8 컨텍스트 파괴 | `addInitScript` 네비게이션 게이팅 | 분석 안정성 |
| 1 | `networkidle` 만으로 렌더링 완료 판단 불가 | 6-signal 복합 readiness gate | 분석 품질 |
| 1 | 환경별 Chrome 경로 불일치 | `CHROME_PATH` env var fallback 체인 | 배포 신뢰성 |
| 2 | `body`/`html` 오탐으로 페이지 전체 숨김 | 루트 요소 명시적 제외 | 오탐율 감소 |
| 2 | 퍼센트 인코딩 불일치 → 중복 노드 | URL 정규화 (`decodeURIComponent` 재인코딩) | 그래프 정확도 |
| 2 | 외부 폰트/CDN 지연으로 크롤 blocking | resource type 기반 차단 | 크롤 속도 |
| 2 | CSS 무한 애니메이션으로 스크린샷 타임아웃 | `animations: 'disabled'` | 스크린샷 안정성 |
| 3 | 크롤 진행 중 스크린샷 확인 불가 | 페이지별 즉시 S3 업로드 + presigned URL | 실시간 UX |
| 3 | JS 오류·4xx 응답 감지 불가 | 커스텀 matcher (console/network) | 오류 감지율 |
| 3 | 크롤 중단 이유·스크린샷 접근 불가 | 한국어 `stopReason` + 즉시 S3 업로드 | 운영 가시성 |
| 4 | 배포 후 selector 변경으로 locator 실패 | CSS→aria→text self-healing | 시나리오 내구성 |
| 4 | 버튼 클릭 race condition | URL 폴링 + exponential backoff | 실행 신뢰성 |
| 4 | `div.cursor-pointer` SPA 컨테이너 크롤 누락 | `isClickableContainer` + `selectorHint` 폴백 | 크롤 커버리지 |
| 4 | 팝업 닫기가 input 값 소실 유발 | `activeElement` 텍스트 입력 가드 | QA 정확도 |
| 5 | 매 크롤마다 재로그인 → rate limit·CAPTCHA | `storageState` 1회 저장·재사용 | 인증 안정성 |
| 5 | ECS 첫 요청 40~60초 콜드스타트 | Pinned Task 상시 예열 | 응답 속도 |

---

## Phase 1 — 기초 안정화 (크롤이 "돌아가게")

> 프로젝트 초기, 가장 먼저 부딪힌 문제들. Playwright를 SPA 환경에서 실행하면 예상치 못한 곳에서 프로세스가 죽거나, "실행은 됐지만 빈 화면을 분석"하는 상황이 반복됐다.

### 1-1. SPA 네비게이션 방어 (`addInitScript`)

최초 크롤 구현 직후, React Router / Vue Router를 사용하는 페이지에서 `page.evaluate()` 도중 SPA 라우터가 `history.pushState`를 호출해 V8 실행 컨텍스트가 파괴되는 오류가 발생했다.  
`page.route()` 기반 차단은 JS 레벨 `pushState`를 잡지 못한다는 점을 발견하고, `addInitScript`로 페이지 JS보다 먼저 실행되는 네비게이션 게이팅 레이어를 주입하는 방식으로 해결.

**도입 패턴**: `spaNavigationDefense.js` — Phase A(ALLOW) / Phase B(LOCK) 2단계 전환

### 1-2. 6-signal 렌더링 완료 판단 (`renderReadinessChecker`)

`waitForLoadState('networkidle')` 만으로 분석을 시작하면 스켈레톤 UI, lazy-load 미완료 상태를 수집하는 문제가 생겼다.  
단일 조건을 버리고 6개 시그널(title 존재, 가시 요소 수, loading 클래스 비율, layout 안정화, networkidle, 콘텐츠 컨테이너 존재)을 복합으로 판단하는 `renderReadinessChecker`를 구현. 최대 8초 이후 degraded 모드로 fallback해 타임아웃으로 전체 크롤이 실패하지 않도록 함.

### 1-3. Chrome 실행 경로 환경 변수화

로컬 개발 환경에서는 정상이나 ECS Fargate 컨테이너에서는 `Failed to launch the browser process` 오류가 발생했다.  
바이너리 경로를 `CHROME_PATH` 환경 변수로 오버라이드할 수 있는 fallback 체인(env → 시스템 Chrome → Playwright 관리 바이너리)을 추가하고, `--no-sandbox` 플래그를 ECS Task Definition에 고정.

---

## Phase 2 — 크롤 품질 향상 (크롤이 "올바르게" 돌아가게)

> 크롤이 돌아가기 시작한 후, "돌아는 가는데 결과가 이상하다"는 문제들. 중복 페이지, 빠진 페이지, 잘못된 요소 차단 등 크롤 품질 자체를 높이는 작업이 필요했다.

### 2-1. 루트 요소 블로커 오탐 제거

블로커 탐지 로직이 viewport 점유율만으로 판단할 때 `body`, `html`, `#root` 같은 루트 요소가 고득점을 받아 페이지 전체가 `visibility: hidden` 처리되는 사고가 발생했다.  
루트 요소 및 최상위 단일 wrapper를 명시적 제외 목록에 추가해 오탐을 완전히 제거.

### 2-2. URL 중복 노드 방지

한글 경로와 퍼센트 인코딩 경로가 다른 URL로 처리되어 동일 페이지를 이중 분석하는 그래프 오염이 발견됐다.  
URL 정규화 함수(`decodeURIComponent` → 재인코딩 → UUID/숫자 세그먼트 치환)를 도입해 dedup key 통일.

### 2-3. 외부 리소스 차단으로 크롤 속도 개선

Google Fonts 등 외부 폰트 CDN이 네트워크 상황에 따라 `networkidle` 조건을 방해하는 경우가 있었다.  
`context.route()`로 font, image, media 리소스 타입과 외부 폰트 CDN URL 패턴을 차단하는 옵션 추가.  
`BLOCK_ANALYSIS_RESOURCES=true` 환경 변수로 전체 비필수 리소스 차단 가능.

### 2-4. CSS 무한 애니메이션 스크린샷 타임아웃

로딩 스피너, 배경 gradient 애니메이션 등 `infinite` keyframe이 있는 페이지에서 `page.screenshot()`이 무한 대기하는 문제가 생겼다.  
모든 스크린샷 호출에 `animations: 'disabled'` 옵션을 추가해 완전히 해결. 시각적 결과물은 첫 프레임 기준으로 동일하게 캡처됨.

---

## Phase 3 — 실시간·관측성 (크롤 내부를 "볼 수 있게")

> 크롤 품질을 높이고 나니, "지금 크롤이 어디까지 갔는지 모른다"는 운영 문제가 남았다. 수분 동안 진행되는 크롤의 진행 상황을 실시간으로 파악할 수 없었고, 중단이 발생해도 이유를 알기 어려웠다.

### 3-1. 스크린샷 실시간 S3 업로드

크롤 완료 후 일괄 S3 업로드하는 방식에서는 진행 중인 페이지 이미지를 수분 뒤에야 볼 수 있었다.  
페이지 분석 완료 즉시 S3 업로드 + presigned URL을 WebSocket 이벤트로 발행하는 방식으로 전환. 실패 시 전체 결과 손실도 방지.

### 3-2. Console / Network 오류 커스텀 Matcher

기능적으로 정상 동작해도 내부 JS 오류(`console.error`)나 4xx/5xx 응답이 있는 상태가 "통과"로 처리됐다.  
step 실행 전 이벤트 리스너로 오류를 누적하고, `toHaveNoConsoleErrors` / `toHaveNoNetworkErrors` matcher로 assertion하는 구조를 추가.

### 3-3. 크롤 중단 이벤트 실시간 전달

도메인 이탈 · 리다이렉트 이탈 · 자동화 차단(CAPTCHA) 3가지 중단 케이스에서 영문 코드나 null만 반환되어 운영 중 원인 파악이 어려웠다.  
각 케이스에 한국어 `stopReason`, `offendingUrl` 필드를 추가하고, 중단 시점 스크린샷을 즉시 S3 업로드해 컨테이너 종료 후에도 접근 가능하게 변경.

| 중단 코드 | 한국어 사유 |
|-----------|------------|
| `stopped_out_of_scope` | `요청 호스트 X가 루트 호스트 Y와 일치하지 않습니다` |
| `stopped_redirect_out_of_scope` | `최종 렌더링된 URL이 X로 이동하여 루트 호스트 Y를 벗어났습니다` |
| `manual_verification_required` | `자동화 차단 감지: CAPTCHA / bot-detection 유형` |

---

## Phase 4 — QA 실행 안정성 (시나리오가 "안정적으로" 실행되게)

> 분석 엔진이 안정화된 뒤, 분석 결과를 실행하는 QA 실행 엔진에서 새로운 문제들이 드러났다. 분석 시점과 실행 시점 사이의 시간차, 그리고 다양한 SPA 패턴 대응이 필요했다.

### 4-1. Self-Healing Locator

배포 사이클마다 CSS class, `data-testid`, aria attribute 등이 변경돼 분석 시점의 selector가 실행 시점에 무효화되는 문제가 지속적으로 발생했다.  
4단계 탐색 폴백(analysisRef → locatorFallback → iframe 스캔 → self-healing)을 구현.  
self-healing은 텍스트·ariaLabel·placeholder·role을 기준으로 live DOM을 재스캔해 점수 기반으로 대안을 자동 선택. `healingUsed: true` 플래그로 locator 품질 저하를 모니터링.

### 4-2. 버튼 클릭 race condition 해결

`Promise.all([page.click(), page.waitForNavigation()])` 패턴이 네비게이션 타이밍에 따라 간헐적으로 실패하거나 무한 대기하는 문제가 있었다.  
`waitForNavigation` 대신 URL 변화 폴링(exponential backoff: 100→200→400→800→1000ms) 방식으로 전환.

### 4-3. SPA `onClick` 컨테이너 크롤 누락

`<a href>` 없이 `navigate()`만 사용하는 `<div onClick>` 패턴에서 크롤러가 해당 경로를 탐지하지 못했다.  
`isClickableContainer` 감지(cursor-pointer 클래스 + 비장식 태그 + interactive visible 조합)와 `selectorHint` 기반 CSS 폴백 로케이터를 추가해 `<a href>` 없는 네비게이션 컨테이너를 크롤 가능하게 만들었다.

### 4-4. 팝업 자동 닫기 — input 포커스 가드

매 QA 스텝 사이에 호출되는 `quickDismissPopups`가 `<input>` 포커스 중에 실행되면서 2가지 부작용이 생겼다.  
자동완성 드롭다운을 팝업으로 오탐해 `Escape`로 입력값을 소실시키거나, 닫기 버튼 클릭이 포커스를 빼앗아 `onBlur` 핸들러가 값을 초기화하는 문제.  
`document.activeElement` 체크로 텍스트 입력 타입(`text`, `search`, `email` 등) 포커스 중에는 dismissal 전체를 건너뜀.

---

## Phase 5 — 인증·인프라 (운영 환경에서 "지속 가능하게")

> 기능이 안정화되고 나서 운영 규모가 커지면서 인증 자동화와 인프라 레벨 문제들이 드러났다.

### 5-1. 이중 인증 크롤 — `storageState` 세션 재사용

익명 크롤 후 인증 크롤을 위해 매 컨텍스트마다 로그인 폼 자동화를 반복하면 rate limit, CAPTCHA 트리거, 세션 중복 생성 문제가 쌓였다.  
최초 1회 `authFlow`로 로그인 후 `storageState`(cookies + localStorage)를 파일 저장, 이후 모든 인증 컨텍스트에 재적용하는 방식으로 변경. 크롤 완료 후 파일과 AWS Secrets Manager 항목을 즉시 삭제해 자격증명 노출 최소화.

### 5-2. ECS Fargate 콜드스타트 — Pinned Task 예열

SQS 메시지로 Worker가 0→1대 스케일아웃될 때 Task 기동에 40~60초가 소요돼 첫 요청 UX가 나빴다.  
ECS 서비스와 별개로 상시 1대를 유지하는 "pinned" Task를 운영해 첫 요청을 즉시 처리.  
실제 스케일아웃은 SQS 메시지 수가 임계치를 넘을 때만 추가 Task를 기동해 비용을 최소화.

---

## 발전 지표 요약

```
Phase 1 — 기초 안정화
  투입 이슈:  3건  (SPA context, render readiness, Chrome path)
  핵심 성과:  크롤 프로세스 생존율 확보

Phase 2 — 크롤 품질 향상
  투입 이슈:  4건  (오탐, URL중복, 폰트blocking, CSS animation)
  핵심 성과:  그래프 정확도 향상 / 크롤 속도 개선

Phase 3 — 실시간·관측성
  투입 이슈:  3건  (S3 즉시업로드, custom matcher, crawl stop 이벤트)
  핵심 성과:  운영 중 크롤 상태 실시간 파악 가능

Phase 4 — QA 실행 안정성
  투입 이슈:  4건  (self-healing, race condition, onClick크롤, popup guard)
  핵심 성과:  시나리오 재실행 성공률 향상

Phase 5 — 인증·인프라
  투입 이슈:  2건  (storageState, ECS cold start)
  핵심 성과:  운영 규모 확장 대응

총 해결 이슈: 16건
```

---

## 핵심 설계 원칙 (경험에서 도출)

| 원칙 | 내용 |
|------|------|
| **Fail-safe degraded mode** | 조건 미충족 시 예외를 던지지 않고 degraded 모드로 진행. 크롤 전체 실패를 방지. |
| **환경 변수 오버라이드** | Chrome 경로, 리소스 차단 여부 등 운영 환경별 다른 값은 코드가 아닌 환경 변수로 제어. |
| **즉시 업로드 원칙** | 결과물(스크린샷, 중단 사유)은 생성 즉시 S3 업로드. 컨테이너 종료 후에도 접근 가능하게. |
| **Guard-before-action** | 팝업 닫기, 네비게이션 게이팅 등 부작용이 있는 작업 전 반드시 상태 체크(activeElement, navigation phase) 수행. |
| **점수 기반 폴백** | Self-healing locator처럼 단일 조건 대신 다중 signal의 점수 합산으로 최적 후보 선택. 오탐을 줄이고 정밀도 향상. |
| **관측성 우선** | 운영 중 발생하는 중단·오류에는 반드시 한국어 사유와 식별 가능한 URL 필드를 포함. |

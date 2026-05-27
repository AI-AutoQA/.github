# AI 파이프라인 — 분석 데이터 수집 및 데이터 전처리

> AutoQA 플랫폼의 AI 파이프라인은 **분석 데이터 수집 → 데이터 전처리 → AI 추론 → 시나리오 생성** 4단계로 동작합니다.  
> 이 문서는 그 중 **1단계(분석 데이터 수집)** 와 **2단계(데이터 전처리)** 를 담당하는 Playwright 기반 웹 분석 엔진의 구조와 핵심 기술을 소개합니다.

---

## 전체 흐름

```
SQS 메시지 수신
    │
    ▼
① 분석 데이터 수집 ──────── Playwright 크롤러
    │  BFS 사이트 탐색
    │  Phase 0 · 1 · 3
    │  이중 인증모드 크롤
    │
    ▼
② 데이터 전처리 ──────────── compactOutput / siteSummary
    │  URL 정규화 · 신뢰도 점수화
    │  functionalPaths 구조화
    │  site-summary.json 생성
    │
    ▼
③ AI 추론  (AI 서버)
    │
    ▼
④ 시나리오 생성  (AI 서버)
```

---

## 1단계 — 분석 데이터 수집

### 1.1 BFS 기반 사이트 탐색

단일 페이지를 분석하는 것이 아니라 사이트 전체를 **BFS(너비 우선 탐색)** 방식으로 순회합니다.  
방문한 페이지는 `hostname + normalizedPath` 를 dedupKey 로 삼아 **그래프 노드**에 등록하여 재방문을 방지합니다.  
탐색 범위는 타깃 도메인(exact host)으로 제한하며, 로그인 리다이렉트처럼 인증 예외 호스트만 허용 목록으로 열어 둡니다.  
최대 탐색 페이지 수는 기존 50에서 **200페이지**로 상향 조정했습니다.

```
BFS frontier
  ├─ 방문 예정 URL 목록
  ├─ dedupKey 중복 확인 → 스킵
  └─ 스코프(도메인) 검사 → 외부 도메인 즉시 차단
```

### 1.2 Phase 0 — 초기 페이지 안정화

본격적인 DOM 추출 이전에 분석을 방해하는 요소를 제거해 **깨끗한 기준선(baseline)** 을 확보합니다.

| 단계 | 처리 방법 |
|------|-----------|
| 모달 · 쿠키 배너 닫기 | "닫기", "건너뛰기" 버튼 우선 클릭 (구매/가입 진행형은 제외) |
| 버튼 없는 오버레이 | Escape 키 전송 |
| 동영상 인트로 | `detectIntroOverlay` → play+seek+ended dispatch → force-hide 순 처리 |
| 자동 재생 미디어 | 강제 일시정지 + 음소거 |
| 잔존 방해 요소 | `visibility: hidden` CSS 강제 적용 |

인트로 오버레이가 **존재하는 경우**에만 제거 프로세스를 실행하고, `pageReadiness.introOverlay` 메타데이터를 리포트에 기록하여 QA 실행기에서 재사용합니다.

### 1.3 Phase 1 — 정적 분석 및 동적 트리거 탐색

페이지 로드 완료 후 DOM과 상호작용 요소를 수집합니다.

**정적 DOM 수집**
- `body` 마운트 후 `networkidle(8s best-effort)` 대기
- DOM 노드 추출 + 의미 없는 wrapper/div 필터링
- 버튼·메뉴·입력·로그인 등 기능별 요소에 색상 라벨링 적용
- 베이스라인 스크린샷 + 어노테이션 스크린샷 캡처

**동적 트리거 탐색**
- `MutationObserver` 설치 후 3초 패시브 관찰로 Auto-Dynamic 영역(광고·캐러셀) 자동 제외
- 트리거 후보(버튼·탭·드롭다운 등) 발굴 → `selectorHint` 중복 제거
- Bounded worker pool로 후보를 **병렬 탐색**
- 각 트리거의 클릭 결과 delta(변경된 요소만) 스크린샷 및 JSON 저장

**분석 품질 분류**
페이지별 분석 결과를 6단계로 분류하여 AI에 전달할 데이터 신뢰도를 표시합니다.

| 상태 | 의미 |
|------|------|
| `analyzed_successfully` | DOM + 링크 정상 추출 |
| `analyzed_partially` | 일부 추출 성공 |
| `screenshot_only_no_dom` | 스크린샷만 존재, DOM 추출 0건 |
| `context_destroyed_mid_analysis` | 분석 중 컨텍스트 파괴 |
| `post_auth_unstable` | 인증 후 DOM 불안정 |
| `failed_analysis` | 분석 전체 실패 |

`screenshot_only_no_dom` 케이스에서는 인증된 `storageState` 가 있을 경우 한 번의 제한적 재시도를 수행합니다.  
SSR 인증 리다이렉트 후 `page.evaluate()` 연속 실패 시에는 **CDP 스냅샷으로 조기 전환**하는 방어 전략이 적용됩니다.

### 1.4 이중 인증모드 크롤

인증이 필요한 서비스의 숨겨진 기능까지 수집하기 위해 **익명 크롤**과 **인증 크롤**을 모두 수행합니다.

```
1. 익명 크롤  → 공개 페이지 및 인증 진입점 수집
2. authBootstrap → 로그인 자동화 (credentials.loginUrl 기반)
3. storageState 저장 (쿠키 + localStorage + sessionStorage)
4. 인증 크롤  → storageState 재사용, 로그인 후 전용 경로 수집
```

인증 모드에서는 저장된 `storageState` 를 QA 실행 단계에서도 재사용하여  
별도의 재로그인 없이 시나리오를 실행할 수 있습니다.

### 1.5 실시간 스크린샷 업로드

페이지 분석 완료 직후 스크린샷을 S3에 **즉시 업로드**하고 presigned URL을 생성합니다.  
이를 실시간 이벤트(Redis → WebSocket) 페이로드에 포함하여 사용자가 크롤 진행 중에도 각 페이지의 스크린샷을 확인할 수 있습니다.

### 1.6 크롤 중단 이벤트

탐색 불가 판정 시 단순 스킵이 아니라 **중단 사유와 컨텍스트를 이벤트로 발행**합니다.

```json
{
  "type": "page_stopped",
  "screenshot": "<presigned-url>",
  "exitUrl": "https://example.com/redirect-target",
  "reason": "외부 도메인으로 이탈 시도 감지"
}
```

---

## 2단계 — 데이터 전처리

수집된 원시 크롤 데이터를 AI가 소비할 수 있는 **최소화·구조화된 형태**로 변환합니다.

### 2.1 URL 패턴 정규화

동일 템플릿의 페이지 URL을 하나의 노드로 병합해 **그래프 중복 노드를 대폭 감소**시킵니다.

```
/product/12345          →  /product/{id}
/user/a1b2c3d4-...      →  /user/{id}
/board/notice/7         →  /board/notice/{id}
```

정규화 규칙은 설정 파일로 외부화하여 사이트별 튜닝이 가능합니다.  
그래프 아이덴티티 생성 로직에도 동일한 정규화를 적용해 신규·기분석·기알려짐 노드를 일관성 있게 분류합니다.

### 2.2 functionalPaths 구조화

단순 URL 목록을 넘어, 각 경로에 **상호작용 메타데이터**를 부착한 `functionalPaths` 를 생성합니다.

```json
{
  "path": "/product/{id}",
  "interactionType": "detail_entry",
  "outcome": "content_display",
  "isSafe": true,
  "requiresAuth": false,
  "confidence": 0.90
}
```

`functionalPaths` 는 AI 시나리오 생성 단계에서 일반 경로보다 **우선 활용**되어  
시나리오의 기능 커버리지와 생성 품질을 높입니다.

### 2.3 요소 신뢰도 점수화 (`scoreCandidateConfidence`)

크롤에서 수집한 상호작용 요소 각각에 **0~1 신뢰도 점수**를 부여합니다.  
텍스트 내용, href 유무, 요소 역할(button/a/nav)을 조합해 산출합니다.

| 요소 유형 | 조건 | 신뢰도 |
|-----------|------|--------|
| 인증 진입 | 명확한 "로그인/회원가입" 텍스트 | 0.95 |
| 인증 진입 | 브랜드명만 존재 (예: Kakao) | 0.55 |
| 내비게이션 | href + 텍스트 모두 존재 | 0.90 |
| 캐러셀 버튼 | 텍스트 없음 | 0.30 |
| 스크롤 상단 | 텍스트 없음 | 0.35 |

`autoScenarioEligible` (신뢰도 ≥ 0.65) 플래그로 자동 생성 가능 후보와 수동 검토 필요 후보를 구분합니다.

### 2.4 시나리오 요약 생성 — `scenario-brief.json`

페이지 단위 분석 결과를 집약한 `scenario-brief.json` 을 생성합니다.

```json
{
  "handoffPolicy": {
    "summaryFirstRequired": true,
    "directScenarioFromBriefAlone": false,
    "autoEligibleCandidateCount": 12,
    "recheckRequiredCandidateCount": 3
  }
}
```

`summaryFirstRequired = true` 로 강제하여 사이트 요약 없이 시나리오를 바로 생성하는 것을 원천 차단합니다.

### 2.5 사이트 요약 생성 — `site-summary.json`

사이트 전체 분석 결과를 **17개 패턴 분류, 9개 시나리오 클러스터**로 집약한 최종 전처리 산출물입니다.  
이 파일이 AI 추론 단계의 **유일한 입력**이 됩니다.

**9개 시나리오 클러스터**

| 클러스터 | 포함 기준 |
|---------|-----------|
| `smoke` | 자동 생성 가능 + 우선순위 높음 |
| `regression` | 자동 생성 가능 중 smoke 외 나머지 |
| `e2e` | 페이지 간 흐름이 있는 경로 |
| `deferredDynamic` | 지연 상호작용 필요 |
| `authSensitive` | 인증 필요 |
| `externalNavigation` | 외부 링크 |
| `contentDiscovery` | 상세/검색 콘텐츠 탐색 |
| `reviewFirst` | medium 신뢰도, 재확인 필요 |
| `needsBriefRecheck` | low 신뢰도, brief 재분석 후 판단 |

**대표 경로 선정 — Round-Robin**  
단순 신뢰도 순 정렬이 아니라 **그룹별 round-robin** 방식으로 대표 경로를 선정합니다.  
동일 페이지 그룹 URL이 쏠리는 편향을 방지하고, 사이트 전체 기능을 균등하게 커버하는 경로 세트를 구성합니다.

---

## 산출물 구조

```
outputs/
└── <runId>/
    ├── graph-snapshot.json      # 페이지 그래프 (노드·엣지)
    ├── final-report.json        # 페이지별 Phase 1 분석 결과
    ├── scenario-brief.json      # 요소 신뢰도 포함 후보 요약
    ├── site-summary.json        # AI 입력용 최종 전처리 산출물
    ├── crawl-graph.html         # 그래프 시각화 (boilerplate edge 약화 표시)
    ├── screenshots/             # 베이스라인 + 어노테이션 스크린샷
    └── trigger-results/         # 트리거별 delta 스크린샷 + 메타
```

---

## 핵심 설계 원칙

| 원칙 | 내용 |
|------|------|
| **AI 입력 최소화** | 전체 HTML 대신 접근성 트리 요약 + 핵심 요소 목록 + 스크린샷만 전달 |
| **신뢰도 게이팅** | 품질 낮은 후보는 AI 입력에서 제외, 수동 검토 플래그 부착 |
| **사이트 요약 우선** | `site-summary.json` 없이 시나리오 생성 불가 강제 |
| **방어적 처리** | DOM 추출 실패 시 CDP fallback, 분석 중단 이벤트 발행 |
| **실시간 가시성** | S3 즉시 업로드 + presigned URL 이벤트로 크롤 진행 중 확인 가능 |

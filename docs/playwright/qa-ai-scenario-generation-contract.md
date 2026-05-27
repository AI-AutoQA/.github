# QA AI Scenario Generation Contract

이 문서는 AI 개발 서버가 `site-summary.json`과 `analysis-summary.json`을 입력으로 받아 QA scenario suite를 생성할 때 지켜야 하는 계약을 정리한다. 목적은 실행 가능한 시나리오만 생성하고, 잘못된 `nodeId`/페이지 조합이나 현재 페이지 URL과 새 탭 URL을 혼동하는 실패를 줄이는 것이다.

## 출력 파일명과 S3 URL

AI가 생성한 QA scenario JSON 파일명은 입력 분석 파일명을 기준으로 만들 수 있다. 현재 샘플 UI는 입력 파일명에서 `.json`을 제거한 뒤 `_qa_scenarios.json`을 붙이므로, `analysis-summary.json`을 입력했다면 `analysis-summary_qa_scenarios.json`이 된다. 확장자 없이 `analysis-summary_qa_scenarios`로 저장해도 S3 object body가 JSON이면 실행 계약상 문제는 없다.

파일명 자체는 Playwright 실행 계약의 필수 조건이 아니다. AI Worker는 생성한 JSON을 S3에 업로드한 뒤 result event의 `suiteS3Url`에 실제 S3 주소를 넣어야 하고, Backend는 그 값을 그대로 Redis pipeline state에 저장한 뒤 `QA_EXECUTE` 메시지의 `suiteS3Url`로 전달한다.

## 입력 해석

- `site-summary.json`은 사이트 전체 패턴 요약이다. 여기의 `functionalPatterns[]`는 바로 클릭할 `nodeId` 목록이 아니다.
- `analysis-summary.json`은 여러 페이지의 요약을 담는 집계 문서다. 최상위 문서를 단일 `scenarioBrief`처럼 처리하지 말고, `pageBriefs[]` 또는 `pageSummaries[]` 안의 페이지별 brief를 선택해서 사용해야 한다.
- 페이지별 brief의 `page.url` 또는 `page.normalizedPath`가 해당 scenario의 첫 `goto.url`과 일치해야 한다.
- scenario step의 `targetRef.nodeId`는 같은 페이지 brief의 `actionCandidates[]`에서만 선택한다.

## 기능 경로 기반 생성

페이지별 brief는 URL 목록만이 아니라 `functionalPaths[]`를 포함할 수 있다. 이 필드는 분석 단계가 발견한 “사용 가능한 기능 경로”를 AI가 바로 시나리오로 확장할 수 있게 정규화한 것이다.

각 `functionalPaths[]` 항목은 다음 의미를 가진다.

- `sourcePage`: 기능이 발견된 페이지와 `authMode`
- `feature`: 대상 기능 후보의 `nodeId`, 텍스트, 목적, 카테고리
- `interaction`: 실행 방식. 예: `click`, `fill → press Enter`, `select`, `check`
- `outcome`: 실행 후 관찰 또는 예상 결과. 예: `route_navigation`, `spa_route_change`, `modal_opened`, `dropdown_opened`, `tab_content_changed`, `search_results_after_enter`
- `evidence.observedByProbe`: Playwright가 안전한 격리 context에서 실제 클릭 probe로 관찰했는지 여부
- `safety.safeForAutoScenario`: 자동 시나리오 생성에 사용해도 되는지 여부

AI는 `functionalPaths[].safety.safeForAutoScenario === true`인 경로를 `actionCandidates[]`보다 우선 사용한다. `safety.dataMutationRisk === true`, `outcome.kind === "no_observable_change"`, 또는 `safeForAutoScenario !== true`인 경로는 자동 시나리오로 생성하지 않는다.

`functionalPaths`를 사용해도 실제 step의 `targetRef.nodeId`는 같은 페이지 brief의 `actionCandidates[]`, `assertionTargets[]`, `elements[]`, 또는 `functionalPaths[].feature.nodeId`에 존재해야 한다. 멀티 페이지 집계에서는 `nodeId`가 `p001_...:node-12`처럼 page prefix를 포함하므로, 한 scenario 안에서 다른 페이지 prefix를 섞으면 안 된다.

`outcome.source === "observed_safe_probe"`이면 관찰된 결과를 우선 신뢰한다. `route_navigation`/`spa_route_change`는 `urlChanged` 또는 `urlChangedOptional` signal과 URL matcher로 검증하고, `modal_opened`/`dropdown_opened`/`tab_content_changed`/`visible_content_changed`는 `domChanged` 또는 `elementVisible` signal을 사용한다. 검색 입력 경로는 `interaction.sequence`의 `fill → press Enter` 순서를 유지한다.

## 인증 모드 분리

분석 요청에 인증 JSON이 없으면 분석 결과는 비로그인 상태만 포함한다. 인증 JSON이 있으면 Playwright 분석은 비로그인 상태를 그대로 유지하면서 로그인 세션을 별도로 만든 뒤 `anonymous`와 `authenticated` 결과를 분리해 제공한다.

- 분석 리포트는 최상위 `authModes`와 페이지/후보별 `authMode`를 포함할 수 있다.
- `authMode` 값은 `"anonymous"` 또는 `"authenticated"`만 사용한다.
- AI는 모든 scenario에 `authMode`를 명시한다. 생략하지 않는다.
- `authMode: "anonymous"` scenario는 `authMode: "anonymous"` 페이지와 후보만 사용한다.
- `authMode: "authenticated"` scenario는 `authMode: "authenticated"` 페이지와 후보만 사용한다.
- `authModes.authenticated.sessionEstablished === true` 또는 `authSession.reuseAvailable === true`일 때만 authenticated scenario를 생성한다.
- authenticated scenario는 이미 로그인된 브라우저 세션에서 시작하므로 로그인 폼 입력/로그인 버튼 클릭 step을 넣지 않는다.
- anonymous scenario는 인증 정보를 사용하지 않는 비로그인 QA 실행으로 검증된다.
- 인증 JSON이 있어도 destructive action, 결제, 민원 제출, 권한 변경처럼 민감한 결과 검증은 계속 금지한다.

## 생성 가능 후보

AI는 click/fill/select/check 같은 action step을 만들 때 아래 조건을 모두 만족하는 후보만 사용한다.

- `actionCandidates[].autoScenarioEligible === true`
- 후보가 포함된 page brief의 `page.normalizedPath`와 scenario의 `preconditions[0].url`이 동일
- `targetRef.nodeId`가 `final-report.json.elements[].qualifiedId` 또는 같은 page brief의 후보 `nodeId`와 일치
- click navigation 검증은 `a`, `button`, input류, 또는 role이 `link/button/tab/menuitem`인 leaf element를 우선 사용
- `div`, `section`, `nav`, `header`, `footer`, `main`, `ul`, `ol` 같은 컨테이너 노드는 action target으로 사용하지 않음
- `form`은 click target으로 사용하지 않음. 검색/제출 흐름은 입력 필드에서 `press Enter`를 우선 사용하고, 명시적인 submit button 후보가 있을 때만 button을 클릭한다.
- `selectorHint`가 `a`, `button`, `form`처럼 너무 넓은 경우에는 text, href, role, page path를 함께 확인해 더 좁은 후보가 없으면 scenario를 만들지 않는다.

사용 금지 예시:

```json
{
  "type": "click",
  "name": "이용약관 링크 클릭",
  "targetRef": { "nodeId": "p001_www_naver_com_:node-39" }
}
```

위 예시는 `node-39`가 실제 이용약관 링크가 아니라 홈 화면의 `#shortcutArea` navigation container이므로 생성하면 안 된다.

## 4insure 실행 결과 기반 보완 규칙

4대사회보험 정보연계센터 134개 시나리오 실행 결과에서 실패 48개 대부분은 서비스 결함이 아니라 생성 규칙과 Playwright Runner 계약 불일치였다. AI 개발 서버는 아래 규칙을 생성 전/후 정규화 단계에서 순서대로 적용해야 한다.

### 0. 반복 UI 흐름은 capability action으로 표현한다

Playwright 공식 POM 권장사항처럼 대규모 테스트에서는 selector와 반복 동작을 한 곳에 모아야 한다. 이 서비스에서는 테스트 파일 POM 클래스 대신 `actionRef.capability`를 사용한다. AI 모델 재학습이 아니라 생성 계약/후처리 규칙으로 처리한다.

- 외부기관 링크: `common.openExternalLink`
- 검색 제출: `common.submitSearch`
- 인증서/보안 프로그램 도움말 문서: `helpDocument.open`
- 관련기관 select: `institution.selectRelatedSite`

```json
{
  "stepId": "s3",
  "type": "action",
  "targetRef": { "nodeId": "p004_www_4insure_or_kr_pbiz_mjon_processIdPswdLgnView_do:node-157" },
  "actionRef": {
    "capability": "helpDocument.open",
    "name": "browserCertificate"
  }
}
```

`action` step은 가능하면 `targetRef.nodeId`를 함께 제공한다. 분석 리포트 후보가 없을 때만 `actionRef.selector`를 사용한다. 외부 페이지 본문, 인증 완료, 민원 제출처럼 서비스가 제어하지 않거나 credential 계약이 필요한 결과는 검증하지 않는다.

### 0-1. 여러 시나리오에서 반복되는 step 묶음은 `flows`로 분리한다

검색, 로그인 진입, 공통 메뉴 이동처럼 같은 step 배열이 여러 scenario에서 반복되면 suite 최상위 `flows`에 한 번만 선언하고 scenario에서는 `type: "flow"` step으로 참조한다. Playwright Worker는 실행 전에 `flowRef`를 실제 step으로 펼치므로, 최종 실행기는 기존 `fill`, `click`, `press`, `action` step만 처리한다.

```json
{
  "flows": {
    "search.basic": [
      {
        "stepId": "fill",
        "type": "fill",
        "targetRef": { "nodeId": "p001_main:node-search" },
        "input": { "valueTemplate": "${flow.keyword}" }
      },
      { "stepId": "submit", "type": "press", "key": "Enter" }
    ]
  },
  "scenarios": [
    {
      "scenarioId": "SEARCH-001",
      "title": "공지사항 검색",
      "data": { "keyword": "자동화" },
      "steps": [
        { "stepId": "search", "type": "flow", "flowRef": "search.basic", "data": { "keyword": "${data.keyword}" } }
      ]
    }
  ]
}
```

생성 규칙:

- `flows` 안의 step도 일반 step 계약을 그대로 지켜야 한다.
- flow 내부에서 호출부 parameter를 참조할 때만 `${flow.xxx}`를 사용한다.
- scenario별 값은 `${data.xxx}`로 남겨두고, flow 호출부 `data`에서 `${data.xxx}`를 `${flow.xxx}`로 연결한다.
- flow nesting은 가능하지만 깊게 중첩하지 않는다. 공통 흐름은 1단계 참조를 기본으로 둔다.

### 0-2. 같은 시나리오를 여러 입력값으로 실행할 때는 `dataSets`를 사용한다

검색어, select option, 기관 코드처럼 같은 step 구조에 데이터만 바뀌는 경우 scenario를 복붙하지 말고 suite 최상위 `dataSets`와 scenario의 `dataSetRef`를 사용한다. 실행 전처리는 row마다 scenario를 생성하고, row data를 `scenario.data`에 병합한다.

```json
{
  "dataSets": {
    "searchKeywords": [
      { "id": "cert", "label": "인증서", "data": { "keyword": "공동인증서" } },
      { "id": "insurance", "label": "보험", "data": { "keyword": "4대보험" } }
    ]
  },
  "scenarios": [
    {
      "scenarioId": "SEARCH-KEYWORD",
      "title": "검색어별 결과 확인",
      "dataSetRef": "searchKeywords",
      "steps": [
        { "stepId": "search", "type": "flow", "flowRef": "search.basic", "data": { "keyword": "${data.keyword}" } }
      ]
    }
  ]
}
```

생성 규칙:

- row에는 사람이 읽을 수 있는 `id` 또는 `label`을 넣는다. 실행 ID는 `SEARCH-KEYWORD__cert`처럼 만들어진다.
- row별 값은 `data` 객체에 넣는다. 단순 row에서는 `id`, `label`, `name`, `title`, `description`, `tags`를 제외한 필드를 data로 사용할 수 있다.
- `dataSetRef`를 쓰는 scenario의 step은 반드시 `${data.xxx}` 키가 row data 또는 scenario 기본 `data`에 존재해야 한다.
- 특정 scenario에만 쓰는 작은 데이터는 `dataRows` inline 배열을 사용할 수 있다. 여러 scenario가 공유하면 `suite.dataSets`를 사용한다.

### 1. 새 탭/팝업은 current page URL로 검증하지 않는다

`target=_blank`, `window.open()`, 외부기관 링크, 인증서 도움말처럼 새 탭이 열리는 액션은 현재 page의 URL이 바뀌지 않는다.

- click step에는 `popupUrlContains` 또는 `popupOpened` signal을 붙인다.
- popup click 다음 step에 `toContainURL` 또는 `toHaveURL` expect를 추가하지 않는다.
- Runner가 popup을 자동으로 닫더라도 signal 결과의 popup URL로 검증이 끝나야 한다.

```json
{
  "type": "click",
  "targetRef": { "nodeId": "p001_www_4insure_or_kr_:node-123" },
  "expectedSignals": [
    { "type": "popupUrlContains", "urlContains": "eng_site", "required": true }
  ]
}
```

### 2. navigation 이후 이전 페이지 `nodeId`를 재사용하지 않는다

click에 `urlChanged` 또는 `urlChangedOptional`이 붙은 뒤에는 이전 페이지의 `targetRef.nodeId`를 다시 쓰는 `capture`, `waitFor`, `expect.targetRef`를 만들면 안 된다.

- 이동 후 검증은 page-level `toContainURL`, `toHaveURL`, body text 조건으로 제한한다.
- 이동 후 새 페이지에서 element 검증이 필요하면 해당 도착 페이지의 analysis brief에서 새 `nodeId`를 선택한다.
- 같은 페이지 텍스트 변화가 아닌데 `capture(text)` 후 `toChangeFromStored`를 쓰지 않는다.

```json
{
  "type": "expect",
  "assertion": {
    "matcher": "toContainURL",
    "value": "selectTmpltRcsmList"
  }
}
```

### 3. 입력 필드는 `maxlength`를 기대값에 반영한다

로그인 ID/PW, 검색어 등 입력 필드에 `maxlength`가 있으면 브라우저가 값을 잘라낸다. 이 경우 `toHaveValue`는 원문 전체가 아니라 잘린 값을 기대해야 한다.

- analysis brief에 `maxLength`가 있으면 `expectedValue = inputValue.slice(0, maxLength)`로 계산한다.
- 긴 입력 안정성 scenario는 실패가 아니라 “maxlength 제한 확인” 목적으로 제목과 기대값을 맞춘다.
- `공인인증서`처럼 오래된 용어는 현재 사이트 문구인 `공동인증서`를 우선 사용한다.

### 4. select는 실제 option value만 사용한다

`select` step의 `input.value`는 화면 label이 아니라 실제 `option.value`여야 한다.

- analysis brief의 `options[]`가 있으면 반드시 그 안의 `value` 중 하나를 사용한다.
- label만 알고 value를 모르면 scenario를 만들지 않거나 `reviewNeeded`로 넘긴다.
- 관련기관/관련사이트 select는 보통 선택만으로 이동하지 않는다. 이동 버튼 후보가 있으면 `select` 뒤에 이동 버튼 `click`과 `popupOpened`/`popupUrlContains`를 추가한다.

### 5. 검색 form은 click하지 않는다

검색 후보가 `tag=form`이거나 `selectorHint="#search"`처럼 form을 가리키면 직접 click하지 않는다.

- 검색 입력 후보를 `fill`한다.
- 제출은 입력 필드에서 `press Enter`를 우선 사용한다.
- 명시적인 검색 버튼 후보가 있을 때만 button을 click한다.

### 6. 변화 비교는 같은 페이지의 같은 요소 변화에만 사용한다

`toChangeFromStored`는 같은 페이지 안에서 같은 요소의 text/aria/value가 바뀌는지 확인할 때만 사용한다.

- URL 변경 비교에는 사용하지 않는다.
- “홈의 사이트맵 링크 텍스트”와 “사이트맵 페이지 제목”처럼 원래 같은 텍스트일 수 있는 값 비교에는 사용하지 않는다.
- 이동 검증은 `toContainURL`, 렌더링 검증은 도착 페이지 heading/body text로 분리한다.

### 7. 인증 시나리오는 authMode를 분리한다

인증 JSON이 없거나 `authSession.reuseAvailable`이 false이면 인증 시나리오는 기존처럼 진입점/안내 페이지까지만 생성한다.

- 인증 진입점 노출 확인
- 인증 필요 안내 페이지 이동 확인
- 인증서/보안 프로그램 도움말 popup 또는 안내 영역 안정성 확인

인증 JSON으로 분석 세션이 만들어진 경우에는 `authMode: "authenticated"` scenario를 생성할 수 있다. 이때 QA 실행은 분석 단계의 storageState를 재사용하므로 로그인 step을 다시 넣지 않는다. 실제 민원 제출, 권한 변경, 결제, 데이터 삭제 같은 민감한 결과 검증은 별도 명시 계약 없이는 생성하지 않는다.

### Playwright Runner 방어 동작

Runner는 잘못 생성된 시나리오가 들어와도 아래 범위에서 실행 결과를 보정하거나 명확한 실패로 분류한다. 단, 이는 생성 규칙을 대체하지 않고 실패 원인 판별을 돕는 방어선이다.

- `form` click target은 실행 전에 실패 처리하고, input `press Enter` 또는 submit button 사용을 안내한다.
- `urlChanged`를 기대했지만 실제로 popup이 열린 click은 popup navigation으로 부분 성공 처리한다.
- click이 timeout/예외로 끝났더라도 popup이 실제로 열렸다면 popup 발생 효과를 부분 성공 처리한다.
- click이 timeout/예외로 끝났더라도 PDF/HWP/Office 문서 요청 또는 다운로드가 감지되면 도움말/문서 열림 효과로 보고 부분 성공 처리한다.
- popup click 직후 current page `toHaveURL`/`toContainURL`이 들어오면 직전 popup URL이 기대값을 포함하는 경우 부분 성공 처리한다.
- 외부 링크 popup이 `chrome-error://` 또는 `:`처럼 불완전한 URL로 끝나도, 클릭 대상의 실제 `href`/텍스트가 기대 도메인을 포함하면 외부 이동 시도 성공으로 보고 부분 성공 처리한다.
- popup이 이미 감지된 click은 잘못된 `urlChanged` signal을 끝까지 기다리지 않고 즉시 popup navigation 부분 성공으로 전환한다.
- bbox fallback이 텍스트 입력칸으로 떨어져 click이 timeout되고 후속 검증이 별도 의미를 갖는 경우, click 자체는 focus 성공을 기준으로 부분 성공 처리해 잘못된 후보 선택이 서비스 실패처럼 보이지 않게 한다.
- bbox fallback이 `ul`, `section`, `div` 같은 컨테이너를 잡으면 내부에서 가장 가까운 `a/button/input/select` 등 실제 action target을 찾아 클릭한다.
- `공인인증서`와 `공동인증서`처럼 인증서 명칭만 바뀌었고 핵심 토큰 `인증서`가 일치하면 텍스트 검증을 부분 성공 처리한다.
- 업무 링크 이동 후 `commMsg.do?msgCd=needCert` 또는 “공동인증서/간편인증 필요” 처리결과 페이지가 나오면 인증 게이트까지 도달한 navigation으로 보고 부분 성공 처리한다.
- expect 대상 `targetRef.nodeId`가 이전 페이지 기준이라 현재 페이지에서 해석되지 않으면, 페이지 변화는 확인됐지만 원하는 요소 검증은 불가능한 상태로 보고 부분 성공 처리한다.
- `toChangeFromStored`가 navigation label과 도착 페이지 heading/title을 비교해 동일 텍스트로 실패한 경우, 최근 URL 전환 후 heading/title류 요소가 캡처 라벨과 일치하면 도착 페이지 확인 성공으로 보고 부분 성공 처리한다.
- `toHaveValue`는 실제 입력값이 `maxlength` 때문에 기대값 prefix로 잘린 경우 부분 성공 처리한다.
- `select`는 label 문자열이나 `nps`, `nhis` 같은 기관 약어가 들어온 경우 실제 `option.value`를 찾아 부분 성공으로 보정한다.
- Validator는 popup 이후 current page URL assertion, `urlChanged` 이후 이전 page `nodeId` 재사용, `form` click target을 실행 전 오류로 잡는다.

## `site-summary` 정책 준수

`functionalPatterns[]`의 정책은 scenario 생성 여부를 결정하는 상위 게이트다.

- `directGenerationAllowed: true`: 같은 page brief에서 eligible 후보를 확인한 뒤 직접 생성 가능
- `directGenerationAllowed: false` + `nextActionForGenerator: fetch_representative_page_brief_then_generate`: `representativePaths[]`에 해당하는 page brief를 먼저 선택하고, 그 page brief의 eligible 후보로 생성
- `nextActionForGenerator: avoid_auto_generation`: 자동 생성 금지. 사람 검토 또는 추가 분석 대상으로만 남김
- `requiresPageBriefRecheck: true`: 반드시 page brief의 candidate confidence, text, targetUrl을 재확인

이번 NAVER 케이스에서 `external-links`와 `detail-entry`는 `directGenerationAllowed:false`이고 대표 경로가 `/NOTICE`였다. 따라서 `/`에서 바로 해당 nodeId를 클릭하는 scenario를 만들면 안 된다.

## 페이지 정합성 규칙

scenario의 현재 페이지는 `goto` step으로 결정된다.

올바른 예시:

```json
{
  "authMode": "anonymous",
  "preconditions": [
    { "stepId": "pre-1", "type": "goto", "url": "/NOTICE" }
  ],
  "steps": [
    {
      "stepId": "s1",
      "type": "click",
      "name": "회사소개 링크 클릭",
      "targetRef": { "nodeId": "p002_www_naver_com_NOTICE:node-33" }
    }
  ]
}
```

잘못된 예시:

```json
{
  "authMode": "anonymous",
  "preconditions": [
    { "stepId": "pre-1", "type": "goto", "url": "/" }
  ],
  "steps": [
    {
      "stepId": "s1",
      "type": "click",
      "targetRef": { "nodeId": "p002_www_naver_com_NOTICE:node-33" }
    }
  ]
}
```

`p002_www_naver_com_NOTICE:node-33`은 `/NOTICE` 페이지에서 수집된 노드이므로 `/`에서 실행하면 안 된다.

## URL 변경과 새 탭 신호

`urlChanged`는 현재 Playwright page의 URL 변경만 의미한다. `target=_blank` 또는 `window.open()`으로 열리는 새 탭은 현재 page URL이 바뀌지 않는다.

- 현재 탭 이동: `expectedSignals: [{ "type": "urlChanged", "required": true }]` 사용 후 `expect.toContainURL` 가능
- 새 탭/팝업 이동: `expectedSignals: [{ "type": "popupUrlContains", "urlContains": "...", "required": true }]` 사용
- 새 탭 URL 일부를 모르면 `expectedSignals: [{ "type": "popupOpened", "required": true }]` 사용
- 새 탭 검증 뒤에는 현재 page에 대한 `toContainURL` expect를 추가하지 않음

새 탭 검증 예시:

```json
{
  "stepId": "s1",
  "type": "click",
  "name": "외부 링크 새 탭 이동 검증",
  "targetRef": { "nodeId": "p002_www_naver_com_NOTICE:node-33" },
  "expectedSignals": [
    { "type": "popupUrlContains", "urlContains": "navercorp.com", "required": true }
  ],
  "resolution": { "preferred": "analysisRef", "strict": true },
  "timeoutMs": 5000
}
```

## 출력 검증 체크리스트

AI 서버는 scenario JSON을 반환하기 전에 아래를 자체 검증해야 한다.

- 모든 `targetRef.nodeId`가 같은 page brief의 eligible `actionCandidates[]`에 존재하는가?
- scenario 첫 `goto.url`과 target node의 원본 page path가 일치하는가?
- `autoScenarioEligible:false` 후보를 action step으로 사용하지 않았는가?
- 컨테이너 노드가 click target으로 사용되지 않았는가?
- `form` 노드가 click target으로 사용되지 않았는가?
- 새 탭 이동을 현재 page의 `urlChanged`/`toContainURL`로 검증하지 않았는가?
- `urlChanged` 이후 이전 페이지의 `targetRef.nodeId`를 다시 쓰는 step이 없는가?
- `fill` 뒤 `toHaveValue` 기대값이 `maxlength`를 반영하는가?
- `select.input.value`가 실제 `options[].value` 중 하나인가?
- `toChangeFromStored`가 navigation/URL 비교 용도로 쓰이지 않았는가?
- 인증 scenario가 실제 로그인 성공/민원 제출까지 요구하지 않는가?
- `site-summary.functionalPatterns[].nextActionForGenerator`가 `avoid_auto_generation`인 패턴에서 scenario를 생성하지 않았는가?
- `timeoutMs`가 과도하게 크지 않은가? Playwright 실행기는 step/default timeout을 최대 `120000ms`로 제한한다.
- 순수 시간 대기(`waitFor.kind=timeout`)가 꼭 필요한가? 필요한 경우에도 `waitFor.ms`는 최대 `60000ms`를 넘기지 않는다.
- CAPTCHA, Turnstile, Cloudflare browser verification 같은 자동화 챌린지를 우회하거나 자동 풀이하는 step을 생성하지 않았는가?

이 체크리스트 중 하나라도 실패하면 해당 scenario는 생성하지 말고, `reviewNeeded` 또는 `skippedReason` 형태로 반환하는 것을 권장한다.
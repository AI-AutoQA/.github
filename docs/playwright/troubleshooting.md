# AutoQA Playwright 트러블슈팅 기록

이 문서는 일반적인 Playwright 사용법을 나열하는 문서가 아니라, AutoQA의 웹 분석 및 QA 실행 과정에서 실제로 수정된 문제와 대응 내용을 Git 이력에 맞추어 정리한 기록이다.

## 작성 기준

- Git 식별자는 저장소 설정의 `user.email=tmdgus0844@gmail.com`을 기준으로 삼았다.
- 같은 이메일로 남은 작성자 표기 `IforCU`은 동일 작업 이력으로 분류했다.
- Playwright 관련 주요 작업 범위는 2026-04-23의 로컬 환경 구성부터 2026-05-22의 엔진 및 학습 개선까지다.
- 기존 문서의 최초 커밋은 `dc42943` (`[docs] - 트러블 슈팅 및 playwright 정리`, 2026-05-21)이다.
- 아래 설명은 커밋 제목, 변경 파일, 현재 코드에서 확인되는 동작만 기록한다. 재현 로그나 수치가 별도로 남지 않은 경우에는 단정적인 장애 원인으로 확대하지 않는다.

## 주요 병합 흐름

| 날짜 | 병합 커밋 | 브랜치 | 확인되는 주제 |
|---|---|---|---|
| 2026-04-23 | `11c2c93` | `pw/S14P31A101-21-playwright로컬환경구성` | Playwright 작업 기반 구성 |
| 2026-04-29 | `7bcd92a` | `pw/S14P31A101-91-playwright엔진개선` | 분석 크롤 및 실행 검증 안정화 |
| 2026-05-06 | `378468a` | `pw/S14P31A101-108-playwright-실시간데이터` | 분석/실행 실시간 이벤트 연결 |
| 2026-05-07 | `a724245` | `pw/S14P31A101-128-playwright-qa개선` | QA 실행 fallback 및 action 처리 |
| 2026-05-08 | `6295b99` | `infra/S14P31A101-141-ecs수동예열` | Playwright worker pinned 서비스 |
| 2026-05-11 | `02ed858`, `6d81af1` | `pw/S14P31A101-161-playwright실시간데이터` | 즉시 스크린샷 및 오류 matcher |
| 2026-05-12 | `6b91c80` | `pw/S14P31A101-176-playwright데이터브리븐재사용` | 데이터 기반 실행 및 flake 대응 |
| 2026-05-14 | `2827fb8` | `pw/S14P31A101-180-playwright리포트개선` | self-healing locator 및 리포트 개선 |
| 2026-05-14 | `94213a5`, `6339805`, `7dd94f2` | `pw/S14P31A101-184-playwright웹-분석-인증기능-개선` | 인증 크롤, 세션 재사용, QA 증적 |
| 2026-05-17 | `4f193a0` | `pw/S14P31A101-204-playwright-web-analyze-improve` | 크롤 중단 사유와 스크린샷 전달 |
| 2026-05-21 | `8d49752` | `develop` -> `master` | 반영 결과 통합 |

## 빠른 참조

| 상황 | 적용된 처리 | 대표 근거 커밋 |
|---|---|---|
| 페이지가 렌더되기 전에 분석됨 | readiness 검사 및 분석 안정화 | `d680b3f` |
| 인증 리다이렉트 뒤 `page.evaluate()`가 반복 실패함 | CDP 스냅샷 조기 전환 | `b46c0e2` |
| 컨테이너에서 Chromium 실행/렌더가 불안정함 | 브라우저 옵션, 폰트/애니메이션 처리 | `74943da`, `cdc48b9` |
| 분석 진행 화면에서 스크린샷을 바로 볼 수 없음 | 페이지별 S3 즉시 업로드와 이벤트 payload | `935f2ee` |
| UI 동작은 성공했지만 콘솔/네트워크 오류를 놓침 | 오류 assertion matcher | `c5b8c78` |
| 배포 뒤 locator가 깨짐 | self-healing resolution | `fbcec22`, `bf61689` |
| 로그인 이후 페이지 분석/실행이 분리되어 재인증됨 | 인증 크롤 및 `storageState` 재사용 | `b670240`, `49d9ff6` |
| 비ASCII URL assertion 또는 클릭 네비게이션이 불안정함 | URL 비교 정규화 및 backoff | `00f1810`, `d012d60` |
| 크롤 중단 원인을 UI에서 파악하기 어려움 | 사유, 이탈 URL, 스크린샷 전달 | `149d536` |
| 첫 worker 실행의 기동 지연을 운영 중 줄여야 함 | 수동 예열용 pinned ECS 서비스 | `52095d1` |

## 1. 분석 시작 시점과 페이지 안정성

### 문제

SPA 또는 지연 렌더링 페이지를 분석할 때 DOM이 충분히 준비되지 않은 상태로 수집되거나, 페이지 컨텍스트가 바뀌는 구간에서 분석이 실패할 수 있다.

### 반영 내용

- `renderReadinessChecker`를 추가해 페이지 준비 상태를 분석 흐름에서 판정하도록 했다.
- 분석 파이프라인, 브라우저 컨텍스트, 크롤 실행 경로를 함께 보강했다.
- SSR 인증 리다이렉트 이후 `page.evaluate()`가 연속 실패하는 경우 CDP 기반 스냅샷으로 조기에 전환하도록 후속 수정했다.

### 확인 위치

- `playwright/src/core/qa-analysis/phase1/renderReadinessChecker.js`
- `playwright/src/core/qa-analysis/phase1/spaNavigationDefense.js`
- `playwright/src/core/qa-analysis/runAnalysis.js`
- `playwright/src/core/qa-analysis/crawl/crawlRunner.js`

### 근거 커밋

- `d680b3f` - `S14P31A101-91 [feat] : QA 분석 크롤 안정성 강화`
- `b46c0e2` - `S14P31A101-184 [fix] : SSR 인증 리다이렉트 후 page.evaluate 연속 실패 시 CDP 스냅샷 조기 전환`
- `2c0d4d2` - `[hotfix] - 웹 분석 timeout 오류 수정`

## 2. 배포 환경의 브라우저 실행과 렌더링

### 문제

로컬 개발 환경과 배포 컨테이너는 Chromium 경로와 런치 조건이 다르며, 외부 폰트 또는 애니메이션이 스크린샷/분석 처리에 영향을 줄 수 있다.

### 반영 내용

- 분석 및 실행 컨텍스트가 `CHROME_PATH` 또는 `PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH`를 사용할 수 있게 구성되어 있다.
- 배포 환경 브라우저 옵션을 정리하는 hotfix가 반영되었다.
- 분석 스크린샷은 `animations: 'disabled'` 옵션을 사용하며, 폰트 리소스 처리도 배포 환경에 맞게 수정되었다.

### 확인 위치

- `playwright/src/core/qa-analysis/browser.js`
- `playwright/src/core/qa-execution/buildExecutionContext.js`
- `playwright/src/core/qa-analysis/runAnalysis.js`
- `playwright/Dockerfile`

### 근거 커밋

- `74943da` - `[hotfix] - 배포 환경 브라우저 옵션 설정`
- `28fb06b` - `[hotfix] - 브라우저 옵션 정리`
- `cdc48b9` - `[fix] - 배포 환경 무한 CSS 로딩 수정 및 Font Resource 설정 업데이트`

## 3. 분석 및 실행 증적의 실시간 전달

### 문제

페이지 분석이 진행 중이어도 결과 스크린샷을 완료 시점까지 기다려야 하면 진행 상태를 확인하기 어렵고, 실패가 발생한 위치를 즉시 파악하기 어렵다.

### 반영 내용

- 페이지 분석 완료 직후 스크린샷을 S3에 업로드하고 `screenshotPresignedUrl`을 분석 페이지 업데이트 이벤트에 포함한다.
- QA 실행에서는 성공/실패 여부와 무관하게 스텝 증적을 저장하도록 확장했다.
- 이후 실행 과정의 업로드 큐 및 실시간 업로드 흐름을 보강했다.

### 확인 위치

- `playwright/src/core/qa-analysis/crawl/crawlRunner.js`
- `playwright/src/infra/s3Artifacts.js`
- `playwright/src/worker/realtimeEvents.js`
- `playwright/src/worker/artifactUploadQueue.js`

### 근거 커밋

- `935f2ee` - `S14P31A101-161 [feat] : 페이지 분석 완료 시 스크린샷 S3 즉시 업로드 및 presigned URL 실시간 이벤트 포함`
- `7751a92` - `S14P31A101-184 [feat] : QA 실행 전체 스텝 스크린샷 저장 - 성공/실패 무관 캡처 및 S3 배포 반영`
- `15cf508` - `[feat] - QA 실행 실시간 업로드`
- `1f6589c` - `[feat] - playwright 화질 변경 및 QA 실행과정 비동기 적용`

## 4. 콘솔 및 네트워크 오류 검증

### 문제

화면 상의 action/assertion만 통과시키면 사용자 흐름 도중 발생한 `console.error` 또는 네트워크 실패를 테스트 결과에서 놓칠 수 있다.

### 반영 내용

- QA matcher 레지스트리에 `toHaveNoConsoleErrors`, `toHaveNoNetworkErrors`를 추가했다.
- 실행 telemetry에서 수집한 콘솔 및 네트워크 오류 수를 assertion 결과로 판정한다.

### 확인 위치

- `playwright/config/qa-matcher-registry.json`
- `playwright/src/core/qa-execution/assertions/builtInMatchers.js`
- `playwright/src/core/qa-execution/steps/executeExpectStep.js`

### 근거 커밋

- `c5b8c78` - `S14P31A101-161 [feat] : toHaveNoConsoleErrors / toHaveNoNetworkErrors assertion matcher 추가`

## 5. Locator 무효화와 실행 호환성

### 문제

분석 시 저장된 대상 정보가 UI 변경 뒤 더 이상 동일한 locator로 해석되지 않으면 QA 실행이 실패한다.

### 반영 내용

- 기존 대상 해석이 실패한 경우 분석 당시 요소 정보와 현재 DOM을 비교하는 `resolveWithHealing` 경로를 추가했다.
- 후속 변경에서 action sequence, analysis reference, locator fallback, healing 로직의 호환성을 함께 개선했다.

### 확인 위치

- `playwright/src/core/qa-execution/target-resolution/resolveTarget.js`
- `playwright/src/core/qa-execution/target-resolution/resolveWithHealing.js`
- `playwright/src/core/qa-execution/target-resolution/resolveAnalysisRef.js`
- `playwright/src/core/qa-execution/target-resolution/resolveLocatorFallback.js`

### 근거 커밋

- `fbcec22` - `S14P31A101-180 [feat] : self-healing locator 추가`
- `bf61689` - `S14P31A101-343 [fix] : QA action sequence 실행 호환성 개선`

## 6. 인증 페이지 분석과 세션 재사용

### 문제

익명 상태에서 볼 수 없는 페이지를 분석하거나 실행할 때, 분석과 QA 실행이 각각 로그인 절차를 반복하면 인증 흐름이 불안정하고 동일한 조건으로 검증하기 어렵다.

### 반영 내용

- 웹 분석에 로그인 자동화와 익명/인증 크롤 모드를 추가했다.
- 인증 성공 시 생성한 `storageState`를 분석 결과와 QA 실행 컨텍스트에서 재사용한다.
- 현재 코드는 인증 정보 자체가 아닌 로그인 후 세션 상태만 파일로 저장하도록 설명하고, worker가 S3의 세션 artifact를 실행 시 내려받아 적용한다.
- 로그인 탐색과 인증 페이지 분류는 후속 수정으로 보강되었다.

### 확인 위치

- `playwright/src/core/qa-analysis/crawl/authBootstrap.js`
- `playwright/src/core/qa-analysis/crawl/authFlow.js`
- `playwright/src/core/qa-analysis/crawl/crawlRunner.js`
- `playwright/src/core/qa-execution/runScenarioSuite.js`
- `playwright/src/worker/jobRunner.js`

### 근거 커밋

- `b670240` - `S14P31A101-184 [feat] : 웹 분석 로그인 자동화 및 이중 인증모드(익명/인증) 크롤 분석 추가`
- `49d9ff6` - `S14P31A101-184 [feat] : QA 실행 시 분석 세션(storageState) 재사용 및 시나리오별 인증모드 적용`
- `e927443` - `S14P31A101-343 [fix] : 웹 분석 로그인 탐색 오류 수정`

## 7. URL assertion과 클릭 기반 페이지 전환

### 문제

비ASCII 문자가 들어간 URL은 브라우저가 percent-encoding 형태로 돌려줄 수 있어 `href` assertion이 false negative를 만들 수 있다. 또한 버튼 클릭 뒤 URL 변화 감지는 타이밍에 민감하다.

### 반영 내용

- URL 비교 유틸을 추가하고 `href` 속성 assertion에서 비ASCII 경로의 encoding 차이를 허용하도록 수정했다.
- 클릭 기반 URL 이동 검사에는 지수 backoff 재시도 처리를 추가했다.
- 탐색 후보 판단은 텍스트 존재 여부 중심에서 클릭 가능한 버튼/대상 중심으로 후속 수정되었다.

### 확인 위치

- `playwright/src/core/shared/urlCompare.js`
- `playwright/src/core/qa-execution/assertions/builtInMatchers.js`
- `playwright/src/core/qa-analysis/runAnalysis.js`

### 근거 커밋

- `00f1810` - `S14P31A101-184 [fix] : href 속성 assertion에서 한글/비ASCII URL percent-encoding 차이로 인한 false negative 수정`
- `d012d60` - `[hotfix] - 버튼 URL 이동 지수 백오프 시도 추가`
- `cf2f1be` - `[fix] - playwright 탐색 로직 변경, 텍스트 유무 -> 버튼 유무`

## 8. 크롤 중단 사유와 디버깅 증적

### 문제

도메인 범위 이탈 또는 자동화 차단으로 페이지 분석이 멈춘 경우, 상태 코드만으로는 운영자가 어떤 URL과 화면에서 중단되었는지 확인하기 어렵다.

### 반영 내용

- `stopped_out_of_scope`, `stopped_redirect_out_of_scope`, `manual_verification_required` 상태를 분리해 처리한다.
- 중단 결과에 한국어 사유, `offendingUrl`, 가능한 경우 `screenshotPresignedUrl`을 전달한다.
- 이탈 또는 challenge 화면은 분석 중단 시점에 즉시 S3 업로드된다.

### 확인 위치

- `playwright/src/core/qa-analysis/runAnalysis.js`
- `playwright/src/core/qa-analysis/crawl/crawlRunner.js`
- `docs/qa-domain-boundary-event.md`

### 근거 커밋

- `149d536` - `S14P31A101-204 [feat] : 크롤 중단 이벤트에 스크린샷·이탈 URL·한국어 사유 추가`
- `c194948` - `S14P31A101-204 [docs] : 웹 분석 페이지 중단 이벤트 API 명세 문서 추가`

## 9. Worker 콜드 스타트 대응

### 문제

요청이 없을 때 worker가 축소되는 운영 방식에서는 최초 요청 시 Playwright worker 기동 시간이 사용자 대기 시간에 포함될 수 있다.

### 반영 내용

- AutoQA Playwright worker를 수동으로 예열할 수 있는 pinned ECS 서비스를 추가했다.
- `infra/scripts/autoqa-worker-pinned.sh`가 `start`, `stop`, `status` 동작을 제공하며 운영자가 필요한 기간에 예열 모드를 켜고 끌 수 있다.
- pinned worker는 상시 운영을 강제하는 설정이 아니라, 응답 시간과 비용을 고려해 선택하는 운영 수단이다.

### 확인 위치

- `infra/scripts/autoqa-worker-pinned.sh`
- `infra/terraform/main/ecs.tf`
- `infra/terraform/main/outputs.tf`

### 근거 커밋

- `52095d1` - `S14P31A101-128 [feat] : autoqa-cluster Playwright worker 수동 예열 pinned ECS service 및 운영 스크립트 추가`
- `6295b99` - `Merge branch 'infra/S14P31A101-141-ecs수동예열' into 'develop'`

## 후속 변경 기록

아래 커밋은 특정 장애 한 건의 수정이라기보다, 위 흐름을 확장하거나 운영 결과를 다듬은 변경으로 분리해 둔다.

| 날짜 | 커밋 | 내용 |
|---|---|---|
| 2026-05-19 | `cc718e8` | Playwright 스크린샷 재시도 전략 및 오류 메시지 수정 |
| 2026-05-21 | `48941d5` | Playwright 엔진 개선 |
| 2026-05-22 | `8a4a2b3` | 웹 분석 Raw DSL 산출물과 인증 탐색 개선 |
| 2026-05-22 | `4ba8512` | Codegen suite 페이지 전환 검증 개선 |
| 2026-05-22 | `b862bcc` | QA 스텝 스크린샷 설명 overlay 추가 |

## 문서 갱신 규칙

새 트러블슈팅 항목은 다음 정보를 함께 남긴다.

1. 사용자에게 보이는 증상 또는 실패 상태 코드
2. 실제로 변경한 처리와 확인할 소스 파일
3. 구현 커밋 해시와, 병합되었다면 병합 커밋 해시
4. 재현 절차나 운영 로그가 존재하는 경우 그 위치

이 규칙을 따르면 문서 내용과 구현 이력이 분리되지 않고, 이후 원인 분석이나 발표 자료 정리에도 그대로 활용할 수 있다.

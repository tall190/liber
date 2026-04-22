---
title: 'AI-ready E2E — AI가 코드를 짜는 시대, 품질 검증은 누가 하나요?'
description: 'AI로 코드 생산성은 올라갔는데, QA는 여전히 사람이 하나요? Playwright가 Selenium보다 E2E에 적합한 이유, data-testid와 aria-busy로 만드는 안정적인 자동화까지 — QA 관점에서 설명합니다.'
pubDate: '2026-04-22'
heroImage: '../../assets/ai-ready-e2e_hero_image.png'
tags: ['AI-ready', 'QA', 'E2E', 'Playwright']
---

마케터가 AI로 카피를 쓰고, 개발자가 AI로 코드를 짜는 시대가 됐어요.

그런데 QA는요? 여전히 사람이 화면을 하나하나 클릭하면서 확인하고 있어요. 기능이 나오는 속도는 AI 덕분에 빨라졌는데, 검증하는 속도는 그대로거든요.

AI가 코드를 만드는 만큼, 코드를 검증하는 것도 자동화해야 하지 않을까요. 이 글은 그 이야기예요.

## AI 시대에 QA 병목이 생기는 이유

개발 속도와 QA 속도의 균형은 전통적으로 이랬어요.

개발자 1명 = QA 1명. 코드가 나오면 검증하는 사이클이 일주일에 한 번 정도 돌았죠.

AI가 들어오면서 이 균형이 깨졌어요. [GitHub의 연구](https://github.blog/2022-09-07-research-quantifying-github-copilots-impact-on-developer-productivity-and-happiness/)에 따르면 Copilot 도입 후 개발자 생산성이 **55% 향상**됐어요. 코드가 더 빠르게 나오고, PR도 더 자주 열려요. 그런데 QA 인력은 그대로예요.

검증 병목이 생기는 거예요.

해결책은 두 가지예요. QA 인력을 늘리거나, QA 자체를 자동화하거나. 실무에서는 대개 두 번째가 더 현실적이에요.

그리고 E2E 테스트가 그 자동화의 핵심이에요.

## E2E 테스트가 뭔가요, 왜 중요한가요

E2E(End-to-End) 테스트는 사용자 시나리오 전체를 자동화해서 검증하는 거예요.

단위 테스트가 함수 하나를 검증한다면, E2E는 "로그인 → 필터 적용 → 결과가 바뀌는지 → 저장이 되는지"까지 실제 브라우저에서 확인해요. 각 부품이 잘 작동하더라도 연결됐을 때 깨지는 문제 — E2E가 배포 전에 그걸 잡아줘요.

AI가 코드를 빠르게 생성할수록 이 레이어가 더 중요해져요. AI가 만든 코드는 동작은 하는데 엣지 케이스에서 깨지는 경우가 많거든요. 그걸 잡아줄 자동화된 눈이 필요해요.

그럼 어떤 도구로 하느냐. 여기서 선택이 중요해요.

![이미지 프롬프트: "A clean diagram showing the software development pipeline: Code (with AI icon) → Unit Tests → E2E Tests → Deploy. E2E layer highlighted with a QA shield icon. Monochrome with navy accent. Minimal, technical."](../../assets/ai-ready-e2e_pipeline_image.png)

## Playwright vs Selenium — E2E 자동화의 선택 기준

E2E 자동화 도구의 대표 주자는 오랫동안 Selenium이었어요. 2004년에 등장해 20년간 업계 표준으로 쓰였죠.

Playwright는 2020년 Microsoft가 만든 도구예요. 출발점부터 달랐어요. Selenium이 브라우저를 "외부에서 제어"하는 방식이라면, Playwright는 브라우저 내부 프로토콜(CDP)에 직접 연결해서 더 빠르고 정확하게 제어해요.

| 비교 항목 | Selenium | Playwright |
|---|---|---|
| **출시** | 2004년 | 2020년 (Microsoft) |
| **환경 설정** | 브라우저 드라이버 별도 설치·버전 관리 필요 | `npx playwright install` 하나로 완료 |
| **비동기 대기** | 고정 대기 또는 수동 `WebDriverWait` 설정 | Locator가 자동 retry — 요소 등장까지 대기 |
| **테스트 격리** | 세션·쿠키 상태가 테스트 간 공유될 수 있음 | 테스트별 독립 브라우저 컨텍스트 |
| **다중 브라우저** | 브라우저별 드라이버 각각 관리 | Chromium · Firefox · WebKit 동시 지원 |
| **공식 권장 선택자** | XPath, CSS 선택자 | `getByTestId()` 공식 API |
| **디버깅** | 에러 메시지로 추론 | Trace Viewer — 스크린샷·네트워크 타임라인 내장 |
| **네트워크 제어** | 외부 프록시 필요 | `page.route()`로 API 응답 직접 모킹 |

Playwright를 선택하는 이유는 세 가지예요.

**첫째, 기다리는 방식이 다르다.** Selenium에서 가장 흔한 실패 원인은 타이밍 문제예요. 요소가 아직 렌더링되기 전에 클릭을 시도해서 실패하는 거죠. 이걸 막으려고 `time.sleep(3)` 같은 고정 대기를 넣으면 CI 환경 속도에 따라 부족하거나 불필요하게 오래 기다려요. Playwright의 Locator는 액션을 실행할 때마다 자동으로 재시도해요. 요소가 등장할 때까지 기다리고, 등장하면 클릭해요. 고정 대기가 필요 없어요.

**둘째, 테스트 격리가 기본이다.** Selenium은 브라우저 세션을 테스트 간에 공유하는 경우가 많아요. 앞 테스트의 로그인 상태나 쿠키가 다음 테스트에 영향을 줘요. Playwright는 테스트마다 새 브라우저 컨텍스트를 만들어요. 앞 테스트가 어떻게 끝나든 다음 테스트는 깨끗한 상태에서 시작해요.

**셋째, `data-testid`를 공식 API로 지원한다.** `page.getByTestId('filter-apply-btn')` — 이 한 줄이 Playwright가 어떤 방향을 지향하는지 보여줘요. 불안정한 XPath나 CSS 선택자 대신, 명시적인 테스트 식별자를 쓰라는 거예요. 이 전략이 E2E 안정성의 핵심이에요.

---

## data-testid — 선택자가 깨지지 않으려면

E2E 테스트가 신뢰를 잃는 가장 흔한 이유는 이거예요.

기능은 그대로인데 테스트가 깨졌어요. 버튼 텍스트가 바뀌거나, CSS 클래스명이 리팩토링됐거나, DOM 구조가 조금 바뀐 거거든요.

```javascript
// ❌ 취약한 선택자들
await page.click('button:has-text("적용")');       // 텍스트 변경 시 깨짐
await expect(page.locator('.filter-result')).toBeVisible(); // 클래스명 변경 시 깨짐
await page.locator('[class*="CampaignRow"]').first().click(); // 구조 변경 시 깨짐

// ✅ testid 기반 — 텍스트·스타일·구조 변경에 무관
await page.getByTestId('filter-apply-btn').click();
await expect(page.locator('[data-testid="campaign-list"]')).toBeVisible();
```

`data-testid`는 UI 변경과 독립적인 테스트 전용 식별자예요. 버튼 문구가 바뀌어도, 클래스명이 바뀌어도, `data-testid`를 건드리지 않으면 테스트는 깨지지 않아요.

data-testid를 어떻게 설계하고 FE 코드에 붙이는지 — 컨벤션, ESLint 강제화, 마이그레이션 전략까지는 FE 관점에서 쓴 글에서 자세히 다뤄요.

> 👉 [AI-ready UI — data-testid 네이밍 컨벤션부터 ESLint 강제화까지](/liber/blog/ai-ready-ui)

여기선 QA 관점에서 중요한 한 가지만 짚을게요. **동적 목록에서는 인덱스 대신 데이터 ID를 써야 해요.**

```javascript
// ❌ 인덱스 기반 — 정렬·필터 변경 시 엉뚱한 행을 검증
await expect(page.locator('[data-testid="campaign-row-0-status"]'))
  .toHaveText('진행 중');

// ✅ 데이터 ID 기반 — 항상 같은 캠페인을 정확히 지정
const TARGET_ID = '42';
await expect(
  page.locator(`[data-testid="campaign-row-${TARGET_ID}-status"]`)
).toHaveText('진행 중');
```

정렬이 바뀌면 `row-0`이 가리키는 캠페인이 달라져요. 테스트가 우연히 통과하거나, 우연히 실패하게 되죠. 데이터 ID 기반 testid가 그 우연을 없애줘요.

그런데 testid를 잘 써도 아직 남는 문제가 있어요. 요소를 찾았는데 데이터가 아직 안 나왔어요.

## aria-busy — 비동기 타이밍을 정확하게 잡는 법

필터를 적용하면 API를 호출하고 결과를 받아와요. 그 시간이 환경마다 달라요. 로컬에서는 빠르고, CI에서는 느릴 수 있어요.

테스트는 요소를 찾은 뒤 바로 값을 확인해요. 그런데 아직 로딩 중이에요.

```javascript
// ❌ 고정 대기 — CI 환경 따라 부족하거나 불필요하게 오래 기다림
await page.click('[data-testid="filter-apply-btn"]');
await page.waitForTimeout(3000);
await expect(page.locator('[data-testid^="campaign-row-"]').first()).toBeVisible();
```

정확한 방법은 "로딩이 끝났다는 신호"를 보고 그때 검증하는 거예요. `aria-busy`가 그 신호예요.

`aria-busy`는 WAI-ARIA 표준 속성이에요. 요소가 업데이트 중이면 `true`, 완료되면 `false`. FE에서 이걸 붙여주면, E2E가 정확한 시점에 검증할 수 있어요.

```javascript
// FE 컴포넌트
<section
  data-testid="campaign-list"
  aria-busy={isFetching}
>
  {results.map(...)}
</section>
```

```javascript
// ✅ aria-busy 기반 대기 — 환경 무관하게 정확한 시점에 검증
await page.click('[data-testid="filter-apply-btn"]');

// 로딩 완료 신호를 기다림
await expect(
  page.locator('[data-testid="campaign-list"]')
).not.toHaveAttribute('aria-busy', 'true');

// 이제 안전하게 결과 확인
await expect(
  page.locator('[data-testid^="campaign-row-"]').first()
).toBeVisible();
```

![이미지 프롬프트: "Two side-by-side comparison diagrams. Left: Timeline showing 'click → fixed wait 3s → check (sometimes fails)'. Right: Timeline showing 'click → aria-busy=true → aria-busy=false → check (always stable)'. Clean, minimal, navy and white."](../../assets/ai-ready-e2e_ariaBusy_image.png)

이게 Selenium에서 Playwright로 넘어오는 것보다 더 중요한 변화예요. 도구의 변화가 아니라 **검증 패턴의 변화**거든요. 고정 대기에서 상태 기반 대기로.

그리고 `aria-busy`는 테스트를 위해 붙이는 거지만, 동시에 스크린 리더가 "지금 로딩 중"이라는 걸 인식하는 접근성 속성이기도 해요. QA 안정성과 접근성을 한 번에 챙기는 거예요.

---

## 실제 적용 결과

이 패턴을 실무에 도입한 전후 비교예요.

| 지표 | 이전 | 이후 |
|---|---|---|
| E2E flaky 비율 | 빈번 (고정 대기 의존) | 거의 없음 (aria-busy 기반) |
| UI 변경 후 테스트 깨짐 | 빈번 (텍스트/클래스 의존) | 거의 없음 (testid 유지 시) |
| 로케일 변경으로 인한 실패 | 발생 | 0건 |
| 신규 E2E 작성 시간 | 2~4시간 | 30분~1시간 |

신규 E2E 작성 시간이 4분의 1로 줄었어요. 선택자를 어떻게 잡을지 고민하는 시간, flaky 원인을 디버깅하는 시간이 사라졌거든요.

QA의 역할은 버그를 찾는 거예요. 그런데 테스트 코드 디버깅에 시간을 쓰고 있다면, 도구가 QA의 발목을 잡는 거예요. Playwright + data-testid + aria-busy는 그 발목을 풀어주는 조합이에요.

AI가 코드 생산성을 올린 만큼, QA도 같은 속도로 따라가야 해요. 더 많이 클릭하는 게 아니라, 클릭을 기계에 맡기고 더 중요한 판단에 집중하는 방식으로.

당신의 E2E 테스트는 AI 속도를 따라갈 준비가 되어있나요?

---

*관련 글: [AI-ready UI — data-testid 네이밍 컨벤션, ESLint 강제화, 마이그레이션 전략](/liber/blog/ai-ready-ui) — FE 개발자 관점에서 보는 같은 주제.*

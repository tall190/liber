---
title: 'AI-ready UI 실전편 — 상태, 숨겨진 데이터, 동적 선택자까지'
description: 'data-testid를 붙이고 나서 AI 에이전트가 진짜 막히는 지점들. aria-busy로 로딩 대기, hidden 검증 데이터, disabled 이유 명시, 디자인 시스템 prop forwarding, 하이브리드 선택자 전략, 애니메이션 제어까지.'
pubDate: '2026-04-22'
heroImage: '../../assets/ai-ready-ui_ai_recommand_image.png'
---

1편에서 `data-testid` 컨벤션을 정리하고 나면, 실제로 AI 에이전트를 붙여보는 순간 또 다른 문제들이 나와요. 버튼은 정확히 찾는데 클릭 후 다음 단계에서 실패하거나, 버튼이 왜 비활성화되어 있는지 AI가 판단 못해서 시나리오가 막히거나, 디자인 시스템 컴포넌트를 거치면 testid가 사라지거나.

이 글은 그 지점들을 다뤄요. 1편이 "AI가 요소를 찾게 만들기"였다면, 실전편은 "AI가 상태를 이해하게 만들기"예요.

## aria-busy — AI에게 "지금 기다려"를 알려주는 방법

가장 흔한 실패 패턴이에요. AI 에이전트가 버튼을 클릭하고 즉시 다음 요소를 찾으려는데, 데이터가 아직 로딩 중이라 요소가 없어요.

```typescript
// AI가 자주 실패하는 시나리오
await page.getByTestId('search-submit-button').click();
await expect(page.getByTestId('result-list')).toBeVisible(); // 즉시 확인 → 실패
```

사람은 "로딩 스피너가 보이니까 기다려야 해"를 시각적으로 판단하지만, AI 에이전트는 접근성 트리를 읽어요. 스피너는 시각 정보라 트리에 의미 있게 올라오지 않거든요.

해결책은 컨테이너에 `aria-busy` 속성을 명시하는 거예요.

```tsx
// ❌ AI가 로딩 완료 시점을 모름
const SearchResults = ({ isLoading, results }) => (
  <section data-testid="search-results">
    {isLoading ? <Spinner /> : <ResultList results={results} />}
  </section>
);

// ✅ aria-busy로 AI에게 대기 시점을 알려줌
const SearchResults = ({ isLoading, results }) => (
  <section
    data-testid="search-results"
    aria-busy={isLoading}
    data-status={isLoading ? 'loading' : 'idle'}
  >
    {isLoading ? <Spinner /> : <ResultList results={results} />}
  </section>
);
```

AI가 작성하는 테스트가 이렇게 달라져요.

```typescript
// aria-busy를 활용한 안정적인 대기 패턴
await page.getByTestId('search-submit-button').click();

// aria-busy가 사라질 때까지 대기
await expect(page.getByTestId('search-results'))
  .not.toHaveAttribute('aria-busy', 'true');

// 이제 결과 확인
await expect(page.getByTestId('result-list')).toBeVisible();
```

`data-status`는 `aria-busy`와 다른 역할이에요. `aria-busy`는 "지금 처리 중"이라는 이진 상태고, `data-status`는 더 세분화된 상태를 나타내요. AI가 "loading", "error", "empty", "idle" 각각에 맞는 시나리오를 만들 수 있게 해줘요.

```tsx
<section
  data-testid="dashboard-metrics"
  aria-busy={isLoading}
  data-status={isLoading ? 'loading' : error ? 'error' : data.length === 0 ? 'empty' : 'idle'}
>
```

## data-reason — disabled의 이유를 DOM에 남기기

버튼이 `disabled` 상태일 때, AI 에이전트는 "이걸 눌러야 하나 말아야 하나" 판단을 못해요.

```tsx
// AI의 관점: 왜 disabled야? 기다려야 해? 뭔가 빠진 게 있어?
<button disabled data-testid="submit-button">제출</button>
```

disabled인 이유가 "API 응답 대기"인지, "필수 항목 미입력"인지, "권한 없음"인지에 따라 AI가 취해야 할 다음 행동이 달라요. 이유를 DOM에 명시하면 AI가 조건 분기 시나리오를 스스로 만들 수 있어요.

```tsx
// ✅ 이유를 명시
<button
  disabled={isSubmitting || !isFormValid}
  data-testid="submit-button"
  data-reason={
    isSubmitting ? 'waiting-api' :
    !isFormValid ? 'validation-error' :
    undefined
  }
>
  제출
</button>
```

AI가 이 구조를 보면 이런 시나리오를 만들어요.

```typescript
// data-reason="validation-error"인 경우: 필수 항목 채우기
const submitBtn = page.getByTestId('submit-button');
const reason = await submitBtn.getAttribute('data-reason');

if (reason === 'validation-error') {
  // 폼 필수 항목 채우고 재시도
  await page.getByTestId('title-input').fill('필수 제목');
} else if (reason === 'waiting-api') {
  // API 응답 대기
  await expect(submitBtn).not.toHaveAttribute('data-reason', 'waiting-api');
}
```

`data-reason`을 도입하면 AI 에이전트가 단순히 "버튼 클릭"이 아니라 "버튼이 활성화될 조건을 충족하고 클릭"하는 시나리오를 생성할 수 있어요.

## hidden 검증 데이터 — 화면에 없지만 AI가 읽어야 하는 값

실무에서 자주 마주치는 패턴이에요. 주문 번호나 내부 ID 같은 값을 화면에는 일부만 보여주는 경우예요.

```tsx
// 화면: "주문 #****89"로 표시
// 실제 API 검증에 필요한 값: "ORDER-2026042201289"
```

AI 에이전트가 결제 완료 후 주문 상태를 API로 검증하려면 전체 주문 번호가 필요해요. 화면엔 없으니 AI가 찾지 못하는 상황이 생겨요.

해결책은 검증용 데이터를 숨겨진 요소로 심어두는 거예요.

```tsx
const OrderConfirmation = ({ order }) => (
  <section data-testid="order-confirmation">
    {/* 사용자에게 보이는 표시 */}
    <p>주문 번호: ****{order.id.slice(-2)}</p>

    {/* AI 검증용 — 화면엔 없지만 접근성 트리에 존재 */}
    <span
      style={{ display: 'none' }}
      data-testid="hidden-order-id"
      aria-hidden="true"
    >
      {order.id}
    </span>
    <span
      style={{ display: 'none' }}
      data-testid="hidden-order-status"
      aria-hidden="true"
    >
      {order.status}
    </span>
  </section>
);
```

`aria-hidden="true"`를 같이 붙이는 게 중요해요. 스크린리더 사용자에게는 숨겨야 하지만, DOM과 접근성 트리에는 존재해서 AI 에이전트가 읽을 수 있어야 해요.

비슷한 패턴으로 날짜, 시간, 임의 ID처럼 매번 바뀌는 값도 래퍼로 감싸두면 좋아요.

```tsx
{/* AI가 이 영역의 텍스트 변경을 '오류'로 오인하지 않도록 */}
<div data-testid="dynamic-content" data-type="timestamp">
  {formatDate(order.createdAt)}
</div>
```

AI 에이전트가 `data-type="timestamp"` 영역은 변동 가능한 값으로 인식하고, assertion 대상에서 제외할 수 있어요.

## Prop Forwarding — 디자인 시스템이 testid를 삼키는 문제

공통 컴포넌트를 쓰다 보면 testid가 중간에 사라지는 문제가 자주 생겨요.

```tsx
// 이렇게 쓰면...
<Button data-testid="submit-creative-button">등록</Button>

// 실제 DOM에서 testid가 없어요
<button class="btn btn-primary">등록</button>
```

`Button` 내부에서 `data-testid`를 명시적으로 전달하지 않으면 사라지거든요. 이게 대형 코드베이스에서 생각보다 자주 발생해요.

가장 간단한 해결책은 나머지 props를 펼쳐서 DOM 요소에 그대로 전달하는 거예요.

```tsx
// ✅ 방법 1: Rest props 전달 (권장)
const Button = ({ children, className, ...rest }) => (
  <button className={`btn ${className ?? ''}`} {...rest}>
    {children}
  </button>
);

// ✅ 방법 2: data-* 속성을 명시적으로 필터링해서 전달
const Button = ({ children, className, ...props }) => {
  const dataProps = Object.fromEntries(
    Object.entries(props).filter(([key]) => key.startsWith('data-'))
  );
  return (
    <button className={`btn ${className ?? ''}`} {...dataProps}>
      {children}
    </button>
  );
};
```

방법 1이 더 간단하지만, 의도하지 않은 props가 DOM에 전달될 수 있어요. 방법 2는 `data-*` 속성만 선별해서 전달해서 더 명시적이에요.

공통 컴포넌트 수정 시 원칙은 하나예요. **P0 테스트에 영향을 줄 수 있는 공통 컴포넌트 마크업 변경은 QA와 사전 공유.** 공통 컴포넌트 하나가 수십 개 테스트에 영향을 주거든요.

## 하이브리드 선택자 전략 — testid가 항상 정답은 아니다

1편에서 "testid를 쓰세요"라고 했는데, 실무에서는 예외가 있어요. 특히 **동적으로 생성되는 항목 + 다국어** 조합일 때요.

실제 사례로 설명할게요. 대시보드의 지표 선택 드롭다운에 "노출", "클릭" 같은 기본 지표와 "CPC", "CTR" 같은 사용자 KPI가 섞여 있는 경우예요.

기본 지표의 testid는 `metric-node-impression`, `metric-node-click` 처럼 언어 독립적으로 만들 수 있어요. 그런데 사용자가 추가한 KPI는 DB에서 동적으로 생성되다 보니 `metric-node-kpi-1`, `metric-node-kpi-2` 같이 번호가 붙어요. 사용자가 KPI를 추가하거나 순서를 바꾸면 번호가 달라지고, testid가 불안정해져요.

```
기본 지표: metric-node-impression → 안정적
사용자 KPI: metric-node-kpi-1 → 사용자 설정에 따라 변동
```

이 경우에는 지표별로 다른 선택 전략이 더 안정적이에요.

| 지표 유형 | 선택 전략 | 이유 |
|---|---|---|
| 기본 지표 (노출/클릭) | testid 사용 | 언어가 바뀌면 텍스트도 바뀜 ("클릭" → "Click") |
| 사용자 KPI (CPC/CTR) | 텍스트 검색 | 약어라 언어 불변. testid가 동적으로 불안정 |

```typescript
// 기본 지표: testid로 안정적으로 선택
await page.getByTestId('metric-node-impression').click();

// 사용자 KPI: 검색창에 텍스트 입력 후 선택
await page.getByTestId('metric-search-input').fill('CPC');
await page.getByRole('option', { name: 'CPC' }).click();
```

선택 전략 결정 기준은 두 가지예요.

- **다국어 환경에서 텍스트가 바뀌는가?** → testid 사용
- **ID가 동적이고 텍스트는 언어 불변인가?** → 텍스트 검색 사용

무조건 testid가 정답이 아니에요. testid가 불안정한 경우엔 오히려 더 나쁜 선택이에요.

## 애니메이션 제거 — E2E flakiness의 주범

CSS 트랜지션 300ms가 E2E 테스트에서 timing 문제를 일으켜요. AI 에이전트가 요소가 이미 화면에 있는데 트랜지션 중이라 클릭이 안 되거나, 사라지는 중인 요소에 접근하려다 실패하거나.

두 군데에서 동시에 처리해야 해요.

**FE: prefers-reduced-motion 전역 CSS**

```css
/* global.css */
@media (prefers-reduced-motion: reduce) {
  *, ::before, ::after {
    animation-delay: -1ms !important;
    animation-duration: 1ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 1ms !important;
    transition-delay: -1ms !important;
    scroll-behavior: auto !important;
  }
}
```

**Playwright: 브라우저 컨텍스트에서 강제 적용**

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    // prefers-reduced-motion: reduce 강제 적용
    contextOptions: {
      reducedMotion: 'reduce',
    },
    launchOptions: {
      args: ['--force-prefers-reduced-motion'],
    },
  },
});
```

이 둘을 같이 적용해야 해요. CSS만 있으면 Playwright가 `reducedMotion` 설정을 안 넘겨주면 무의미하고, Playwright config만 있으면 FE에서 CSS로 애니메이션을 제어하지 않으면 효과가 없어요.

이 설정 하나로 timing 관련 flaky test가 절반 이상 줄어요. 경험적으로 E2E 간헐적 실패의 주요 원인 중 하나가 애니메이션이거든요.

## 실전 적용 체크리스트

이 글에서 다룬 패턴을 컴포넌트 작성 시 체크하는 기준으로 요약하면 이래요.

```
비동기 컨테이너
  ├─ aria-busy={isLoading}
  └─ data-status="loading | idle | error | empty"

버튼 disabled
  └─ data-reason="waiting-api | validation-error | no-permission"

API 검증이 필요한 숨겨진 값
  └─ <span style={{ display: 'none' }} data-testid="hidden-{field}" aria-hidden="true">

공통 컴포넌트
  └─ {...rest} 또는 data-* 필터링으로 prop forwarding

동적 리스트 선택자
  ├─ 언어 변경되는 텍스트 → testid
  └─ 언어 불변 약어 + 동적 ID → 텍스트 검색

E2E 환경
  ├─ prefers-reduced-motion CSS 전역 적용
  └─ playwright.config.ts reducedMotion: 'reduce'
```

---

data-testid를 도입하고 나서 AI 에이전트 자동화가 잘 안 된다면, 대부분 여기서 다룬 패턴 중 하나가 빠져 있는 거예요. 요소를 찾는 건 1편, 상태를 이해하게 만드는 건 실전편 — 두 가지가 함께 있어야 AI 에이전트가 사람처럼 화면을 다룰 수 있어요.

---

*이전 글: [AI-ready UI 1편 — AI 에이전트가 당신의 화면을 탐색하는 방식](/liber/blog/ai-readable-ui)*

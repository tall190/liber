---
title: 'AI-ready UI — 당신의 서비스는 AI와 함께 일할 준비가 되어있나요?'
description: 'AI가 개발 속도를 높이고 QA까지 자동화하려면, UI가 먼저 AI에게 읽혀야 한다. data-testid 컨벤션부터 동적 테이블, 다국어 지원, AI 도구 활용법까지.'
pubDate: '2026-04-21'
heroImage: '../../assets/ai-ready-ui_hero_image.png'
---

AI 코딩 어시스턴트를 쓰다 보면 신기한 순간이 있어요. 복잡한 비즈니스 로직을 설명했더니 거의 정확한 코드를 내놓고, 리팩토링을 부탁하면 제가 생각했던 방향보다 더 깔끔하게 정리해주더라고요.

근데 분명히 막히는 순간도 있어요. 특히 UI 자동화와 테스트를 부탁할 때요.

## AI가 막히는 곳

"이 저장 버튼 클릭했을 때 토스트 메시지 뜨는지 테스트 써줘."

AI는 코드를 읽기 시작해요. 컴포넌트 트리를 파악하고, 클릭 핸들러를 추적하고, 무슨 일이 벌어지는지 추론하려 해요. 근데 버튼에 이름이 없어요.

```tsx
// AI가 이해하기 어려운 구조
<div className="btn btn-primary" onClick={handleClick}>
  저장
</div>
```

AI 입장에서 이 코드는 모호해요. `handleClick`은 다른 버튼들도 쓰고 있고, `btn-primary`는 스타일 정보일 뿐이고, 텍스트 "저장"은 다음 릴리스에 "저장하기"로 바뀔 수도 있거든요. AI는 이 버튼이 정확히 어떤 역할을 하는지 확신하기 어려워요.

반면 이렇게 쓰면 달라져요.

```tsx
// AI가 의도를 바로 이해하는 구조
<button data-testid="save-draft-button" onClick={handleSaveDraft}>
  저장
</button>
```

`data-testid="save-draft-button"` — 이 속성 하나가 AI에게 명확한 계약을 만들어줘요. "이 버튼은 임시저장 버튼이에요. 텍스트가 바뀌어도, 스타일이 바뀌어도, 이 이름으로 찾아요."

이게 **AI-ready UI**의 시작이에요.

![No testid vs With testid — AI가 UI를 이해하는 방식의 차이](../../assets/ai-ready-ui_ai_understand_image.png)

그럼 이게 실제 개발 흐름에서 어떤 차이를 만드는지 볼게요.

## 개발 단계에서 UI 구조가 만드는 차이

식별자가 있는 컴포넌트는 AI와의 협업 자체가 달라져요.

새 기능을 만들 때 AI에게 맥락을 설명하는 시간이 줄어들어요. "이 폼은 광고 소재 등록 폼이고, 제출 버튼은 `submit-creative-button`이야" — 코드 자체가 이미 그 설명을 담고 있거든요. AI는 컴포넌트 트리를 읽고 의도를 파악해요.

시맨틱 HTML을 더하면 구조가 훨씬 명확해져요.

```tsx
<main data-testid="creative-upload-page">
  <section data-testid="creative-form-section">
    <form data-testid="creative-upload-form">
      <input data-testid="creative-title-input" />
      <input data-testid="creative-url-input" />
      <button data-testid="submit-creative-button">등록</button>
    </form>
  </section>
</main>
```

AI는 이 구조를 보고 "광고 소재 업로드 페이지에, 소재 등록 폼이 있고, 제목 입력과 URL 입력, 제출 버튼이 있다"는 맥락을 파악해요. 개발자가 따로 설명 안 해도요.

다국어 서비스라면 이 효과가 더 커져요.

## 다국어 서비스에서 testid가 더 중요한 이유

영어, 한국어, 일본어를 지원하는 서비스를 만든다고 생각해보세요. 텍스트 기반으로 UI를 탐색하는 코드는 언어가 바뀔 때마다 깨져요.

```tsx
// ❌ 언어가 바뀌면 깨지는 방식
const saveButton = page.getByText('저장');       // 한국어
const saveButton = page.getByText('Save');        // 영어
const saveButton = page.getByText('保存');        // 일본어
```

세 가지를 따로 관리하거나, 조건 분기를 만들거나, 둘 다 끔찍하죠.

`data-testid`는 언어와 무관해요. 텍스트는 바뀌어도 이름은 그대로예요.

```tsx
// ✅ 언어가 바뀌어도 동일하게 작동하는 방식
<button data-testid="save-draft-button">
  {t('common.save')}  {/* 한국어: 저장 / 영어: Save / 일본어: 保存 */}
</button>
```

```typescript
// 언어에 관계없이 항상 같은 코드
await page.getByTestId('save-draft-button').click();
```

i18n 라이브러리로 텍스트를 추상화한 것처럼, testid로 UI 식별도 추상화하는 거예요. 다국어 지원을 처음부터 고려한다면 선택이 아니라 필수예요.

동적으로 생성되는 요소도 마찬가지예요.

## 동적 테이블과 리스트에 testid 심기

"동적으로 생성되는 테이블 행에는 어떻게 testid를 붙이나요?" — 이 질문을 자주 받아요.

정답은 **데이터의 고유 식별자를 함께 쓰는 것**이에요. 인덱스(0, 1, 2...)는 정렬이나 필터링을 적용하면 바뀌지만, 데이터 ID는 변하지 않거든요.

```tsx
// ❌ 인덱스 기반 — 정렬하면 깨짐
{campaigns.map((campaign, index) => (
  <tr data-testid={`campaign-${index}-row`}>
    <td>{campaign.name}</td>
  </tr>
))}

// ✅ 데이터 ID 기반 — 정렬/필터링에 안전
{campaigns.map((campaign) => (
  <tr data-testid={`campaign-${campaign.id}-row`}>
    <td data-testid={`campaign-${campaign.id}-name`}>{campaign.name}</td>
    <td data-testid={`campaign-${campaign.id}-status`}>{campaign.status}</td>
    <td>
      <button data-testid={`campaign-${campaign.id}-edit-button`}>수정</button>
      <button data-testid={`campaign-${campaign.id}-delete-button`}>삭제</button>
    </td>
  </tr>
))}
```

이렇게 하면 AI가 "캠페인 ID 42번의 수정 버튼을 클릭해줘"라는 지시를 정확하게 수행할 수 있어요.

컨테이너 레벨에도 testid를 붙여두면 더 좋아요.

```tsx
<div data-testid="campaign-list">
  <div data-testid="campaign-list-header">
    <button data-testid="campaign-sort-name-button">이름순</button>
    <button data-testid="campaign-sort-date-button">날짜순</button>
    <input data-testid="campaign-search-input" />
  </div>
  <table data-testid="campaign-table">
    {/* 행들 */}
  </table>
  <div data-testid="campaign-pagination">
    <button data-testid="campaign-prev-page-button">이전</button>
    <span data-testid="campaign-current-page">1</span>
    <button data-testid="campaign-next-page-button">다음</button>
  </div>
</div>
```

정렬 → 특정 항목 선택 → 수정 → 저장 같은 복잡한 시나리오도 AI가 처음부터 끝까지 수행할 수 있어요. 그리고 이게 QA 자동화로 바로 연결돼요.

## 개발이 빨라지면, QA가 병목이 된다

AI 코딩 어시스턴트 덕분에 개발 속도가 올라갔어요. 하루에 만들 수 있는 기능의 수가 늘었죠. 근데 그러면 다음 질문이 자연스럽게 따라와요.

**검증 속도도 함께 올라갔나요?**

개발자가 10개 기능을 만들어도, QA가 3개밖에 검증 못하면 전체 속도는 3개로 수렴해요. 개발 속도가 빨라질수록, 병목은 QA로 이동해요.

해결책은 QA도 AI와 함께하는 거예요. E2E 테스트를 AI가 작성하고, 회귀 케이스를 AI가 추적하고, 릴리스 전 검증을 AI가 실행하고. 근데 여기서도 같은 문제가 생겨요.

## QA AI도 같은 곳에서 막힌다

Playwright, Cypress 기반으로 AI가 테스트를 작성하려면, UI 요소를 정확하게 찾아야 해요. 텍스트 기반 탐색은 취약하고, CSS 클래스는 스타일 리팩토링에 무너지고, 시각 인식은 레이아웃 변경에 취약해요.

`data-testid`가 있으면 달라지죠.

```typescript
// AI가 작성하는 안정적인 E2E 테스트
test('캠페인 임시저장', async ({ page }) => {
  await page.goto('/campaigns/new');
  await page.getByTestId('campaign-title-input').fill('여름 프로모션');
  await page.getByTestId('save-draft-button').click();
  await expect(page.getByTestId('draft-saved-toast')).toBeVisible();
  await expect(page.getByTestId('draft-saved-toast'))
    .toContainText('저장되었습니다');
});
```

디자인이 바뀌어도, 텍스트가 바뀌어도, 언어가 바뀌어도 — 이 테스트는 흔들리지 않아요. 개발 단계에서 붙인 `data-testid`가 QA 단계의 자동화를 가능하게 해줘요. 같은 이름이 두 단계를 연결하는 거예요.

여기에 하나 더 챙기면 더 강해져요.

## 접근성 트리가 AI의 눈이다

`data-testid`와 함께 주목할 게 하나 더 있어요. **ARIA와 시맨틱 HTML**도 AI가 UI를 이해하는 경로예요.

[ProofSource.ai의 연구](https://proofsource.ai/2026/01/agent-browser-the-accessibility-first-approach-to-browser-automation/)에 따르면, AI 에이전트가 전체 DOM 대신 접근성 트리(Accessibility Tree)만 읽도록 했을 때 컨텍스트 토큰이 **93% 절감**됐어요. AI는 의미 단위로 요약된 구조를 훨씬 잘 이해하거든요.

스크린리더를 위해 작성한 ARIA 속성이 AI 에이전트에게도 정확한 인터페이스가 돼요. [BrowserStack의 Playwright MCP 분석](https://www.browserstack.com/guide/modern-test-automation-with-ai-and-playwright)도 같은 결론이에요 — ARIA 속성이 AI 테스트 생성 정확도에 직접 영향을 줘요.

```tsx
// ARIA + testid 함께 사용하기
<dialog
  data-testid="confirm-delete-modal"
  aria-labelledby="modal-title"
  aria-describedby="modal-description"
>
  <h2 id="modal-title" data-testid="modal-title">삭제 확인</h2>
  <p id="modal-description" data-testid="modal-description">
    이 캠페인을 삭제하면 복구할 수 없어요.
  </p>
  <button data-testid="confirm-delete-button" aria-label="삭제 확인">
    삭제
  </button>
  <button data-testid="cancel-delete-button" aria-label="취소">
    취소
  </button>
</dialog>
```

접근성이 좋은 UI는 AI에게도 좋은 UI예요. 두 목표가 같은 방향을 가리켜요.

"근데 기존 코드베이스가 수백 개 컴포넌트인데 어떻게 하나요?" — 당연한 걱정이에요. 다행히 AI 도구 자체가 이 작업을 도와줄 수 있어요.

## AI 도구로 더 쉽게 적용하기

**Claude Code / Copilot에 이렇게 물어보세요:**

```
이 컴포넌트 파일에 있는 모든 인터랙티브 요소(button, input, a, select)에
data-testid를 추가해줘. 네이밍 규칙은:
- 버튼: {action}-button
- 입력 필드: {field}-input
- 링크: {destination}-link
- 셀렉트: {field}-select
이미 data-testid가 있는 요소는 건드리지 마.
```

**ESLint 커스텀 룰로 자동 강제화:**

새로 작성하는 코드에 testid를 빠뜨리지 않도록 lint 룰을 만들 수 있어요.

```javascript
// .eslintrc.js — 인터랙티브 요소에 testid 필수화
module.exports = {
  rules: {
    'local/require-testid': ['warn', {
      elements: ['button', 'input', 'select', 'a'],
      attribute: 'data-testid',
    }]
  }
}
```

**PR 리뷰에 AI를 활용:**

CodeRabbit, GitHub Copilot Code Review 같은 AI 리뷰 도구에 프롬프트를 추가하면, PR마다 testid 누락을 자동으로 잡아줘요.

```
코드 리뷰 시 다음을 확인해주세요:
- button, input, select 요소에 data-testid가 있는지
- testid 네이밍이 {action}-{type} 패턴을 따르는지
- 동적 리스트의 경우 인덱스 대신 데이터 ID를 사용하는지
```

이런 식으로 초기 정착만 시켜두면, 팀 전체가 자연스럽게 따라오게 돼요.

## AI-ready UI: 전 사이클의 공통 언어

팀 내에서 네이밍 컨벤션을 정해두면, AI가 예측 가능하게 작동해요.

```
{page}-page                    # 페이지 루트
{section}-section              # 영역 구분
{entity}-list                  # 리스트 컨테이너
{entity}-{id}-row              # 테이블 행 (데이터 ID 사용)
{action}-button                # 액션 버튼
{field}-input                  # 입력 필드
{field}-select                 # 셀렉트 박스
{destination}-link             # 내비게이션 링크
{status}-{type}-badge          # 상태 표시
{name}-modal                   # 모달
{name}-toast                   # 토스트 알림
```

패턴이 예측 가능할수록, AI의 추론 오류가 줄어들어요. 개발 단계에서 AI가 더 정확한 코드를 제안하고, QA 단계에서 AI가 더 안정적인 테스트를 작성해요.

이건 테스트 규칙이 아니에요. **AI와 함께 일하기 위한 공통 언어**예요.

![FE Development → data-testid → QA Automation — 전 사이클을 연결하는 공통 언어](../../assets/ai-ready-ui_testid_image.png)

## 사람도 같이 이익이다

AI-ready UI는 AI만을 위한 게 아니에요.

명확한 `data-testid`는 사람인 개발자가 테스트를 쓸 때도 편해요. 시맨틱 HTML은 스크린리더 접근성을 높이고, 다국어 서비스에서 텍스트와 식별자를 분리해줘요. 명확한 컴포넌트 경계는 코드 리뷰를 쉽게 만들고요.

AI가 읽기 좋은 UI는 사람도 읽기 좋은 UI예요.

---

AI가 개발 속도를 높였다면, 다음은 검증이에요. 개발부터 QA까지 AI와 함께 하려면, UI가 먼저 AI에게 읽혀야 해요.

당신의 서비스는 AI와 함께 일할 준비가 되어있나요?

---

*다음 글: [AI-ready Marketing — 당신의 브랜드를 AI에게 설명할 준비가 되었나요?](/liber/blog/geo-for-brands) — 같은 원리의 다른 이야기. 이번엔 개발팀이 아닌 브랜드가 주인공이에요.*

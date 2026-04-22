---
title: 'AI-ready UI 실전편 — 동적 testid 설계, 계층 구조, aria-busy 배치 전략'
description: '동적으로 생성되는 UI에서 testid를 어떻게 조합하는가. 텍스트가 아닌 위치를 특정하는 계층 구조 설계. aria-busy를 어디에 두어야 AI가 정확히 대기하는가. 3-4년차 FE 개발자를 위한 실전 패턴.'
pubDate: '2026-04-22'
heroImage: '../../assets/ai-ready-ui_ai_recommand_image.png'
---

1편에서 `data-testid` 컨벤션을 잡고 나면, 실제 시나리오를 돌려보는 순간 세 가지 지점에서 막혀요.

첫째, 동적으로 생성된 항목에 testid를 어떻게 붙이냐. 둘째, 텍스트가 맞아도 엉뚱한 위치에서 매칭되는 문제. 셋째, 로딩 대기 시점을 AI가 정확히 잡지 못하는 문제. 이 세 가지를 순서대로 다룰게요.

## 동적 UI의 testid 설계 — "동적이라서 testid 못 쓴다"는 틀렸다

동적으로 생성되는 항목에 testid를 못 쓴다는 건 잘못된 전제예요. 세 가지 패턴으로 대부분의 동적 케이스를 처리할 수 있어요.

**패턴 1: API에서 받은 ID로 조합**

생성 시 서버에서 ID를 리턴받으면, 그 ID를 testid에 조합하면 돼요. 인덱스와 달리 데이터가 추가·삭제·정렬돼도 안정적이에요.

```tsx
{campaigns.map((campaign) => (
  <tr data-testid={`campaign-${campaign.id}-row`}>
    <td data-testid={`campaign-${campaign.id}-name`}>{campaign.name}</td>
    <td data-testid={`campaign-${campaign.id}-status`}>{campaign.status}</td>
    <button data-testid={`campaign-${campaign.id}-edit-button`}>수정</button>
  </tr>
))}
```

AI가 "ID 42번 캠페인의 상태가 ACTIVE인지 확인해줘"를 처리할 때 `campaign-42-status`를 정확히 찾아요.

**패턴 2: 정렬 순서 기반 인덱스**

"목록에서 첫 번째 항목"을 찾아야 할 때는 인덱스가 오히려 정확한 표현이에요. "최신순 정렬 후 첫 번째 캠페인"이라는 시나리오에서 `campaign-list-item-0`은 의미가 명확해요.

```tsx
{campaigns.map((campaign, idx) => (
  <tr
    data-testid={`campaign-${campaign.id}-row`}
    data-list-index={idx}
  >
    ...
  </tr>
))}
```

`data-testid`는 ID 기반으로, `data-list-index`는 정렬 순서용으로 분리해서 두 목적 모두 처리할 수 있어요.

**패턴 3: 화면 표시 + API 검증이 동시에 필요한 경우**

화면엔 일부만 보여주고 (예: `#****89`), API 검증에는 전체 값이 필요한 경우 hidden 요소로 분리해요.

```tsx
const OrderRow = ({ order }) => (
  <tr data-testid={`order-${order.id}-row`}>
    {/* 화면 표시용 */}
    <td data-testid={`order-${order.id}-number-display`}>
      #{order.number.slice(-4)}
    </td>

    {/* AI 검증용 — 화면에 안 보이지만 DOM에 존재 */}
    <span
      style={{ display: 'none' }}
      data-testid={`order-${order.id}-full-number`}
      aria-hidden="true"
    >
      {order.number}
    </span>
  </tr>
);
```

화면 표시 값과 검증 값이 다를 때, 숨겨진 요소가 AI의 assertion 앵커가 돼요.

## 텍스트 매칭의 진짜 문제 — 위치를 특정해야 한다

"저장" 버튼을 클릭하는 테스트를 생각해보세요. 화면에 "저장"이라는 텍스트가 두 군데 있다면 어느 쪽이 실행될지 모르죠.

```typescript
// ❌ 텍스트로만 찾으면 의도하지 않은 요소 클릭 가능
await page.getByText('저장').click();

// ❌ testid를 붙였어도 계층 없이 쓰면 같은 문제
await page.getByTestId('save-button').click();
// → 모달 안의 save-button인지, 폼 아래 save-button인지 불명확
```

텍스트 매칭에서 진짜 중요한 건 "이 텍스트가 내가 원하는 위치에 있는 텍스트인지"예요. 텍스트 자체가 아니라 **맥락** 이 기준이거든요. testid가 계층 구조를 덮고 있어야 AI가 정확한 위치를 알 수 있어요.

```tsx
{/* 계층 구조에 testid를 심으면 위치 특정이 가능해짐 */}
<dialog data-testid="confirm-delete-modal">
  <section data-testid="modal-content">
    <h2 data-testid="modal-title">삭제 확인</h2>
    <p data-testid="modal-description">이 캠페인을 삭제하면 복구할 수 없어요.</p>
    <div data-testid="modal-actions">
      <button data-testid="modal-confirm-button">삭제</button>
      <button data-testid="modal-cancel-button">취소</button>
    </div>
  </section>
</dialog>
```

이 구조에서 AI는 "modal-actions 안에 있는 confirm-button"을 찾아요. 같은 이름의 버튼이 페이지 어딘가에 있어도 스코프가 잡혀 있어서 헷갈리지 않아요.

```typescript
// 계층으로 스코프를 잡은 탐색
const modal = page.getByTestId('confirm-delete-modal');
await modal.getByTestId('modal-confirm-button').click();
```

테이블도 마찬가지예요. 행 → 셀 → 버튼 계층이 있어야 "이 행의 수정 버튼"이라는 의미가 명확해져요.

```tsx
<table data-testid="campaign-table">
  <tbody data-testid="campaign-table-body">
    {campaigns.map((campaign) => (
      <tr data-testid={`campaign-${campaign.id}-row`}>
        <td data-testid={`campaign-${campaign.id}-name`}>{campaign.name}</td>
        <td data-testid={`campaign-${campaign.id}-actions`}>
          <button data-testid={`campaign-${campaign.id}-edit-button`}>수정</button>
          <button data-testid={`campaign-${campaign.id}-delete-button`}>삭제</button>
        </td>
      </tr>
    ))}
  </tbody>
</table>
```

`campaign-42-row` 아래에서 `campaign-42-edit-button`을 찾는 건, 같은 이름의 요소가 몇 개 있어도 혼동이 없어요. 계층이 맥락을 담당하거든요.

![계층 구조와 다이얼로그에 testid를 심는 패턴](../../assets/ai-ready-ui_testid_image.png)

## aria-busy — 상위 컨테이너 하나에만 두는 이유

`aria-busy`를 잘못 배치하면 오히려 타이밍 잡기가 더 어려워져요. 흔한 실수는 로딩 중인 요소마다 붙이는 거예요.

```tsx
{/* ❌ 여러 곳에 분산 — AI가 어느 걸 기준으로 기다려야 할지 모름 */}
<section>
  <div aria-busy={isHeaderLoading} data-testid="header-section">...</div>
  <table aria-busy={isTableLoading} data-testid="campaign-table">...</table>
  <div aria-busy={isPaginationLoading} data-testid="pagination">...</div>
</section>
```

여러 `aria-busy`가 각자 다른 타이밍에 해제되면, AI 에이전트는 "어느 것이 끝났을 때 다음 단계로 가야 하나"를 판단하기 어려워요.

```tsx
{/* ✅ 상위 컨테이너 하나에만 — 명확한 대기 기준점 */}
<section
  data-testid="campaign-dashboard"
  aria-busy={isHeaderLoading || isTableLoading || isPaginationLoading}
  data-status={isLoading ? 'loading' : 'idle'}
>
  <div data-testid="header-section">...</div>
  <table data-testid="campaign-table">...</table>
  <div data-testid="pagination">...</div>
</section>
```

AI 에이전트 입장에서 `campaign-dashboard`의 `aria-busy`가 `false`가 되는 순간이 "이 화면의 로딩이 완료됐다"는 신호예요. 명확한 대기 기준점이 하나 있어야 시나리오가 안정적이에요.

```typescript
// 상위 컨테이너의 aria-busy를 기준으로 대기
await page.getByTestId('campaign-dashboard').click(); // 탐색 트리거
await expect(page.getByTestId('campaign-dashboard'))
  .not.toHaveAttribute('aria-busy', 'true'); // 로딩 완료 대기
// 이후 안전하게 내부 요소 탐색
await expect(page.getByTestId('campaign-42-row')).toBeVisible();
```

어디에 `aria-busy`를 두어야 하는지 결정하는 기준은 간단해요. "이게 끝나면 AI가 다음 단계로 가도 된다"는 경계가 어디인지를 생각하면 돼요. 그 경계가 `aria-busy`를 두는 위치예요.

## disabled의 이유를 DOM에 남기기

버튼이 `disabled`일 때 AI는 왜 비활성인지 알 수 없어요. 기다려야 하는지, 다른 입력이 필요한지 판단이 안 되거든요.

```tsx
{/* ❌ AI가 이유를 모름 */}
<button disabled data-testid="submit-button">제출</button>

{/* ✅ 이유가 명시됨 */}
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

`data-reason="waiting-api"`를 보면 AI가 "API 응답을 기다렸다가 다시 시도"하는 시나리오를 만들어요. `data-reason="validation-error"`면 "필수 입력을 채운 후 재시도"라는 흐름을 잡을 수 있어요.

## 애니메이션 — flaky test의 주범을 제거하는 방법

E2E 간헐적 실패의 주요 원인 중 하나가 CSS 트랜지션이에요. FE와 테스트 설정 두 곳에서 동시에 제어해야 해요.

```css
/* global.css */
@media (prefers-reduced-motion: reduce) {
  *, ::before, ::after {
    animation-duration: 1ms !important;
    transition-duration: 1ms !important;
  }
}
```

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    contextOptions: { reducedMotion: 'reduce' },
    launchOptions: { args: ['--force-prefers-reduced-motion'] },
  },
});
```

CSS만 있으면 Playwright가 `reducedMotion` 설정을 전달하지 않아서 무의미하고, config만 있으면 CSS에서 애니메이션을 제어하지 않으면 효과가 없어요. 두 곳이 함께 작동해야 해요.

## 실전 의사결정 체크리스트

```
동적 리스트 항목
  ├─ 서버에서 ID를 받는가?
  │    └─ YES → data-testid={`entity-${id}-row`}
  ├─ 정렬 순서가 중요한가?
  │    └─ YES → data-list-index={idx} 병행
  └─ 화면 표시값 ≠ API 검증값?
       └─ YES → 숨겨진 span에 전체값 + aria-hidden="true"

계층 / 위치 특정
  ├─ Dialog, Modal → data-testid 필수
  ├─ Section, Sidebar → data-testid 필수
  └─ Table row → entity-{id}-row로 스코프

aria-busy 배치
  └─ 상위 컨테이너 하나에만
       "이게 끝나면 다음 단계로 가도 된다"는 경계

disabled 상태
  └─ data-reason="waiting-api | validation-error | no-permission"

E2E 환경
  ├─ prefers-reduced-motion CSS 전역 적용
  └─ playwright.config.ts reducedMotion: 'reduce'
```

---

testid를 도입했는데 AI 에이전트 자동화가 불안정하다면, 대부분 계층 구조 누락이거나 `aria-busy` 배치 문제예요. 요소를 찾는 건 1편, 정확한 위치와 타이밍을 잡는 건 이 글 — 두 가지가 함께 있어야 AI 에이전트가 사람처럼 화면을 다뤄요.

---

*이전 글: [AI-ready UI 1편 — AI 에이전트가 당신의 화면을 탐색하는 방식](/liber/blog/ai-readable-ui)*

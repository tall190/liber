---
title: 'AI-ready UI — AI 에이전트가 당신의 화면을 탐색하는 방식'
description: 'AI 에이전트가 DOM을 읽는 원리부터 UI 식별자 선택 비교, 동작하는 ESLint 룰, production 최적화, 레거시 마이그레이션 전략까지. 3년차 이상 FE 개발자를 위한 실전 가이드.'
pubDate: '2026-04-21'
heroImage: '../../assets/ai-ready-ui_hero_image.png'
---

Playwright MCP로 AI 에이전트가 브라우저를 직접 조작하게 해봤을 때였어요. 버튼 클릭 하나가 계속 실패하는데, 이상한 건 그 버튼이 화면에 멀쩡히 보인다는 거였어요. AI는 버튼을 "찾지 못했다"고 했고, 저는 "왜?"라는 질문 앞에서 잠깐 멈췄어요.

AI가 화면을 사람처럼 본다는 가정이 틀렸던 거예요.

## AI 에이전트는 DOM 전체를 읽지 않는다

Playwright MCP, Claude Computer Use, Cursor의 UI 탐색 — 이것들이 화면을 어떻게 읽는지 실제로 살펴보면, 대부분 **접근성 트리(Accessibility Tree)** 를 기반으로 동작해요.

접근성 트리는 브라우저가 시각 정보를 걷어내고 "의미 단위"로 재구성한 DOM의 요약본이에요. 스크린리더가 읽는 바로 그 구조예요. 시각 스타일, 레이아웃, z-index 같은 건 여기 없어요. "이게 버튼이고, 레이블은 이거고, 클릭 가능하다" — 이 정도만 있어요.

[ProofSource.ai 연구](https://proofsource.ai/2026/01/agent-browser-the-accessibility-first-approach-to-browser-automation/)에서 AI 에이전트가 전체 DOM 대신 접근성 트리만 읽게 했을 때 컨텍스트 토큰이 93% 줄었어요. 전체 DOM이 얼마나 노이즈로 가득한지를 역설적으로 보여주는 수치예요.

문제는 **접근성 트리에 정보가 없으면, AI는 추측하거나 실패한다는 거예요.**

```tsx
// 접근성 트리에서 이렇게 보여요
<div className="btn btn-primary" onClick={handleClick}>
  저장
</div>

// 접근성 트리 출력: role=generic, name="저장"
// → "저장"이라는 텍스트를 가진 클릭 가능한 무언가. 정체 불명.
```

`div`는 시맨틱 역할이 없어요. 클릭 가능한지도 알 수 없어요. 접근성 트리에 역할이 없으면, AI 에이전트는 이 요소를 버튼으로 인식하지 못하거나 신뢰도가 낮은 추론에 의존해요.

## UI 식별 방법 4가지의 실전 비교

"AI가 요소를 찾는 방법"은 선택지가 몇 가지 있어요. 각각 트레이드오프가 달라요.

| 방법 | 스타일 변경 | i18n | 리팩토링 | AI 탐색 정확도 |
|---|---|---|---|---|
| CSS class | ❌ | ✅ | ❌ | ❌ |
| 텍스트 내용 | ✅ | ❌ | ⚠️ | ⚠️ |
| aria-label | ✅ | ✅ | ✅ | ✅ |
| data-testid | ✅ | ✅ | ✅ | ✅ |

**CSS class는 스타일과 식별자를 혼용한 거예요.** `btn-primary`가 `btn-cta`로 바뀌는 순간 테스트가 깨지고, AI 에이전트도 찾지 못해요. Tailwind로 전환하면 클래스 자체가 사라지죠.

**텍스트 기반 탐색은 다국어 서비스에서 바로 무너져요.** "저장" → "Save" → "保存" — 언어별 분기를 테스트 코드에 넣거나, 모든 언어로 테스트를 중복 작성해야 해요. 유지보수 부채가 바로 쌓여요.

**aria-label은 접근성을 위한 거예요.** 스크린리더 사용자에게 의미를 전달하는 목적이 있어요. 자동화 식별자로 남용하면 접근성 의미가 오염돼요 — 예를 들어 `aria-label="campaign-42-edit-button"` 같은 label은 스크린리더 사용자한테 아무 의미가 없어요.

**data-testid는 자동화를 명시적으로 표현하는 속성이에요.** 스타일과 무관하고, 텍스트와 무관하고, 접근성 트리에도 올라가요. 목적이 명확해서 남용 여지가 없어요.

결론: **aria는 접근성 목적으로, data-testid는 자동화 목적으로 각각 써야 해요.** 두 가지는 역할이 달라서 같이 쓰는 게 맞아요.

```tsx
// 역할이 다른 두 속성
<button
  data-testid="confirm-delete-button"   // 자동화 앵커
  aria-label="캠페인 삭제 확인"           // 스크린리더 안내
  aria-describedby="delete-warning"
>
  삭제
</button>
```

그럼 실제 코드베이스에 어떻게 적용할지로 넘어갈게요.

![No testid vs With testid — AI가 UI를 이해하는 방식의 차이](../../assets/ai-ready-ui_ai_understand_image.png)

## 네이밍 컨벤션 — 충돌 방지가 핵심이다

data-testid를 처음 도입하면 금방 충돌이 생겨요. `submit-button`이 광고 등록 폼에도 있고, 캠페인 생성 폼에도 있고, 설정 페이지에도 있어요.

**컨텍스트를 앞에 붙이는 게 원칙이에요.**

```
// ❌ 컨텍스트 없음 → 충돌 발생
submit-button
delete-button
title-input

// ✅ 컨텍스트 포함 → 충돌 없음
creative-form-submit-button
campaign-42-delete-button
ad-group-title-input
```

전체 패턴 체계:

```
{page}-page                      # 페이지 루트
{section}-section                # 영역 구분
{entity}-list                    # 리스트 컨테이너
{entity}-{id}-row                # 테이블 행 (인덱스 ❌, 데이터 ID ✅)
{entity}-{id}-{field}            # 셀 안의 데이터
{context}-{action}-button        # 액션 버튼
{context}-{field}-input          # 입력 필드
{context}-{field}-select         # 셀렉트
{destination}-link               # 내비게이션 링크
{name}-modal                     # 모달
{name}-toast                     # 토스트
```

동적 리스트에서 인덱스 기반 testid는 정렬이나 필터링 한 번이면 바로 깨져요.

```tsx
// ❌ 인덱스 기반 — 정렬하면 깨짐
{campaigns.map((campaign, index) => (
  <tr data-testid={`campaign-${index}-row`}>

// ✅ 데이터 ID 기반 — 정렬/필터링에 안전
{campaigns.map((campaign) => (
  <tr data-testid={`campaign-${campaign.id}-row`}>
    <td data-testid={`campaign-${campaign.id}-name`}>{campaign.name}</td>
    <td data-testid={`campaign-${campaign.id}-status`}>{campaign.status}</td>
    <button data-testid={`campaign-${campaign.id}-edit-button`}>수정</button>
  </tr>
))}
```

## ESLint로 강제화하기 — 실제 동작하는 코드

이 부분에서 잘못된 예시를 많이 봤어요. `local/require-testid`를 eslintrc에 선언만 해서는 작동 안 해요. 커스텀 룰은 플러그인 형태로 등록해야 해요.

ESLint v9 flat config 기준:

```js
// eslint.config.js
const requireTestId = {
  meta: {
    type: 'suggestion',
    messages: {
      missing: '<{{element}}> 에 data-testid가 필요해요.',
    },
  },
  create(context) {
    const INTERACTIVE = new Set(['button', 'input', 'select', 'textarea', 'a']);
    return {
      JSXOpeningElement(node) {
        const name = node.name.name;
        if (!INTERACTIVE.has(name)) return;

        const hasTestId = node.attributes.some(
          (attr) =>
            attr.type === 'JSXAttribute' &&
            attr.name?.name === 'data-testid'
        );

        if (!hasTestId) {
          context.report({
            node,
            messageId: 'missing',
            data: { element: name },
          });
        }
      },
    };
  },
};

export default [
  {
    plugins: {
      local: { rules: { 'require-testid': requireTestId } },
    },
    rules: {
      'local/require-testid': 'warn',
    },
  },
];
```

ESLint v8(legacy config)이라면 `eslint-plugin-local` 패키지를 써서 같은 룰을 등록할 수 있어요.

**단, 무조건 warn으로 시작하세요.** error로 하면 레거시 파일이 많은 코드베이스에서 CI 전체가 즉시 막혀요.

## Production에 data-testid를 남겨도 될까

자주 나오는 질문이에요. 결론부터: **대부분의 경우 괜찮아요.**

data-testid는 HTML attribute예요. 기능에 영향 없고, 보안 취약점도 없어요. 100개 요소 기준 gzip 이후 추가되는 용량은 1KB 미만이에요.

다만 제거하고 싶다면 두 가지 방법이 있어요.

**방법 1: babel-plugin-react-remove-properties (CRA, 커스텀 babel 환경)**

```bash
npm install --save-dev babel-plugin-react-remove-properties
```

```js
// babel.config.js
module.exports = {
  env: {
    production: {
      plugins: [
        ['react-remove-properties', { properties: ['data-testid'] }],
      ],
    },
  },
};
```

**방법 2: Vite 환경에서 커스텀 플러그인**

```ts
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig(({ mode }) => ({
  plugins: [
    mode === 'production' && {
      name: 'remove-testid',
      transform(code: string, id: string) {
        if (!/\.[jt]sx$/.test(id)) return;
        return code.replace(/\s+data-testid="[^"]*"/g, '');
      },
    },
  ].filter(Boolean),
}));
```

저는 제거 안 하는 팀이 더 현실적이라고 봐요. 제거 플러그인 자체가 테스트 대상이 되고 (production 빌드에서 정말 제거됐는지 검증해야 하므로), 관리 포인트가 늘어요. 용량 절감 대비 유지보수 비용이 더 크거든요.

## 레거시 코드베이스 마이그레이션 — 200개 컴포넌트를 어떻게 하나

"새로운 컴포넌트에만 적용하면 되지 않나요?" — 이렇게 시작하면 6개월 뒤에 testid가 절반만 있는 어중간한 코드베이스가 돼요. AI 에이전트가 testid 있는 영역만 신뢰하고 나머지는 텍스트 추론으로 폴백하는 상황이 만들어져요.

현실적인 마이그레이션 전략은 세 단계예요.

**1단계: AI로 파일 단위 일괄 마이그레이션**

Claude나 Cursor에게 파일 하나씩 넘기면서 이렇게 부탁해요:

```
이 컴포넌트의 모든 button, input, select, a 요소에 data-testid를 추가해줘.
규칙:
- 이미 data-testid가 있는 요소는 건드리지 마
- 네이밍 패턴: {context}-{action}-button, {context}-{field}-input
- 동적 리스트의 경우 인덱스 대신 데이터 ID 사용 (e.g. campaign.id)
- 페이지명은 파일 경로에서 추론해줘
```

파일 1개당 2-3분이면 돼요. 리뷰하면서 병렬로 진행하면 하루에 수십 개 처리할 수 있어요.

**2단계: Boy Scout Rule 도입**

ESLint warn 룰을 켜고, 팀에 공지해요: "본인이 수정한 파일의 testid 경고는 같은 PR에서 해결하기." 강제가 아니라 자율로 시작하되, PR 리뷰에서 체크포인트로 삼는 거예요.

**3단계: 핵심 Flow 우선 완료 후 CI error 전환**

주요 사용자 흐름 (로그인, 핵심 업무 기능)의 testid가 완성되면, ESLint를 warn → error로 전환해요. 이후 신규 파일에서 testid가 빠지면 CI에서 막혀요.

## 실제로 뭐가 달라지나

접근성 트리가 풍부해지면 AI 에이전트의 동작이 눈에 띄게 달라져요.

```typescript
// testid 없는 환경 — AI의 selector 추론
await page.locator('button:has-text("저장")').click();
// → 한국어 환경에서만 동작. 텍스트 변경에 취약.

// testid 있는 환경 — 안정적인 selector
await page.getByTestId('creative-form-save-button').click();
// → 언어, 텍스트, 스타일 변경에 무관하게 동작.
```

AI 에이전트가 시나리오 단위로 동작할 때 차이가 더 커요.

```typescript
// AI가 실행하는 복잡한 시나리오
test('광고 소재 임시저장 후 재편집', async ({ page }) => {
  await page.getByTestId('creative-title-input').fill('여름 프로모션');
  await page.getByTestId('creative-save-draft-button').click();
  await expect(page.getByTestId('draft-saved-toast')).toBeVisible();

  // 목록으로 나갔다가 다시 들어옴
  await page.getByTestId('creative-list-link').click();
  await page.getByTestId(`creative-${draftId}-edit-button`).click();

  await expect(page.getByTestId('creative-title-input'))
    .toHaveValue('여름 프로모션');
});
```

testid 체계가 없는 코드베이스에서는 이 테스트가 텍스트 추론과 CSS 셀렉터로 뒤덮여요. 디자인 변경 한 번이면 절반이 깨지고, 고치는 데 testid 붙이는 것보다 더 많은 시간이 들어요.

![FE Development → data-testid → QA Automation — 전 사이클을 연결하는 공통 언어](../../assets/ai-ready-ui_testid_image.png)

## 어디까지 해야 충분한가

기준이 없으면 끝이 없어요. 실용적인 기준으로는 이렇게 잡아요.

**필수 (반드시):**
- 모든 `button`, `input`, `select`, `textarea`
- 페이지 루트와 주요 섹션 컨테이너
- 동적 리스트의 행과 핵심 셀

**권장 (중요한 UI):**
- 모달, 토스트, 드롭다운
- 페이지네이션 컨트롤
- 상태 표시 배지 (상태값이 테스트 assertion 대상인 경우)

**불필요:**
- 순수 표시용 텍스트 (`<p>`, `<span>`)
- 아이콘 (인터랙션 없는 경우)
- 레이아웃 컨테이너 (의미 없는 `<div>` 중첩)

---

AI가 코드를 잘 만들어주는 건 이미 됐어요. 다음 단계는 AI가 완성된 UI를 직접 검증하고, 회귀를 잡고, QA 싸이클 전체에 참여하는 거예요. 근데 그러려면 AI가 UI를 읽을 수 있어야 해요.

testid는 "테스트를 위한 설정"이 아니에요. AI와 함께 일하는 FE 코드베이스의 기본 인프라예요. 지금 붙이지 않으면, 나중에 AI가 "왜 이 버튼을 못 찾냐"고 할 때 다시 돌아오게 돼요.

컨벤션을 잡은 다음엔 실전 문제가 남아요. 동적으로 생성되는 항목엔 testid를 어떻게 조합하는지, 텍스트가 맞아도 잘못된 위치에서 매칭되는 문제, AI가 로딩 완료 시점을 정확히 잡게 하는 `aria-busy` 배치 전략까지 — 다음 글에서 다뤄요.

[AI-ready UI 실전편 → 동적 testid 설계, 계층 구조, aria-busy 배치 전략](/liber/blog/ai-ready-ui-advanced)

---

*[AI-ready Marketing — 당신의 브랜드를 AI에게 설명할 준비가 되었나요?](/liber/blog/geo-for-brands) — 개발팀이 아닌 브랜드가 주인공인 이야기.*

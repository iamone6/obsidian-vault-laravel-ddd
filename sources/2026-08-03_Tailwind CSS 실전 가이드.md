# Tailwind CSS 실전 가이드 (v4.3 기준)

> 대상: CSS 속성을 보면 "아, 이건 여백/색상/정렬이구나" 정도는 읽히지만, CSS 아키텍처를 직접 설계해 본 경험은 많지 않은 백엔드 개발자.
> 목표: 개념 이해 → 문법 습득 → 프로덕션에서 유지보수 가능한 코드로 넘어가기.
> 기준 버전: **Tailwind CSS v4.3.x** (2026년 7월 기준 최신 안정판). 예제는 Laravel/Blade 위주지만, 개념은 프레임워크와 무관합니다.

---

## 목차

1. [Tailwind가 해결하려는 문제](#1-tailwind가-해결하려는-문제)
2. [설치 (Laravel + Vite 중심)](#2-설치-laravel--vite-중심)
3. [클래스 이름 읽는 법 — CSS ↔ Tailwind 대응](#3-클래스-이름-읽는-법--css--tailwind-대응)
4. [레이아웃: Flex와 Grid](#4-레이아웃-flex와-grid)
5. [Variant: 상태·반응형·다크모드](#5-variant-상태반응형다크모드)
6. [실전 컴포넌트 예제](#6-실전-컴포넌트-예제)
7. [설정: CSS-first 방식 (`@theme`, `@utility`)](#7-설정-css-first-방식-theme-utility)
8. [프로덕션 핵심 ①: 컴포넌트화 전략](#8-프로덕션-핵심--컴포넌트화-전략)
9. [프로덕션 핵심 ②: 동적 클래스 함정](#9-프로덕션-핵심--동적-클래스-함정)
10. [다크모드 실전](#10-다크모드-실전)
11. [접근성 체크리스트](#11-접근성-체크리스트)
12. [빌드·배포·성능](#12-빌드배포성능)
13. [팀 협업과 유지보수](#13-팀-협업과-유지보수)
14. [한국어 사이트에서 자주 필요한 것들](#14-한국어-사이트에서-자주-필요한-것들)
15. [v3 → v4 차이 (인터넷 예제 볼 때 주의)](#15-v3--v4-차이-인터넷-예제-볼-때-주의)
16. [자주 하는 실수 모음](#16-자주-하는-실수-모음)
17. [치트시트](#17-치트시트)
18. [학습 경로와 참고 자료](#18-학습-경로와-참고-자료)

---

## 1. Tailwind가 해결하려는 문제

### 1.1 기존 방식의 마찰 지점

전통적인 CSS 작업은 이렇게 흘러갑니다.

```html
<div class="card">
  <h2 class="card__title">제목</h2>
</div>
```

```css
.card { padding: 1rem; border-radius: 0.5rem; background: #fff; box-shadow: 0 1px 2px rgb(0 0 0 / .05); }
.card__title { font-size: 1.125rem; font-weight: 600; }
```

이 방식에서 반복적으로 발생하는 문제는 대략 이렇습니다.

| 문제 | 설명 |
|---|---|
| **네이밍 비용** | `.card`, `.card--compact`, `.card__title--muted`... 이름 짓는 데 시간이 들고, 팀마다 규칙이 다릅니다. |
| **전역 스코프** | CSS는 기본적으로 전역입니다. `.title`을 수정하면 어디가 깨질지 확신하기 어렵습니다. |
| **삭제 불가 CSS** | HTML에서 마크업을 지워도, CSS 파일의 규칙은 남습니다. "이거 지워도 되나?"를 판단할 수 없어서 파일이 계속 커집니다. |
| **컨텍스트 전환** | HTML과 CSS 파일을 왔다 갔다 해야 합니다. |
| **일관성 붕괴** | 누구는 `padding: 15px`, 누구는 `16px`. 디자인 토큰이 강제되지 않습니다. |

이건 개발자 역량 문제가 아니라 CSS라는 언어의 구조적 특성입니다. 대규모 프로젝트에서 CSS가 커지는 속도가 HTML보다 빠른 이유이기도 합니다.

### 1.2 유틸리티 우선(utility-first) 접근

Tailwind는 "한 가지 일만 하는 작은 클래스"를 미리 만들어 두고, HTML에서 조합하게 합니다.

```html
<div class="p-4 rounded-lg bg-white shadow-sm">
  <h2 class="text-lg font-semibold">제목</h2>
</div>
```

- `p-4` → `padding: 1rem`
- `rounded-lg` → `border-radius: 0.5rem`
- `shadow-sm` → 미리 정의된 그림자

즉, **CSS를 새로 작성하는 대신 미리 정의된 디자인 토큰을 조합**합니다.

### 1.3 얻는 것과 잃는 것

Tailwind는 만능이 아니고, 명확한 트레이드오프가 있습니다. 도입 전에 팀과 공유해 둘 필요가 있는 부분입니다.

**얻는 것**

- 이름 짓지 않아도 됩니다.
- 스타일 수정 범위가 해당 엘리먼트로 한정됩니다. 다른 화면이 깨질 걱정이 줄어듭니다.
- 마크업을 지우면 스타일도 같이 사라집니다. CSS 파일이 무한정 커지지 않습니다.
- 값이 스케일(`p-1`, `p-2`, `p-4`...)로 제한되어 일관성이 유지됩니다.
- 실제 사용된 클래스만 빌드에 포함되므로 최종 CSS가 작습니다(보통 10~50KB 미만, gzip 후 더 작음).

**잃는 것 / 감수해야 하는 것**

- HTML이 길어집니다. 클래스가 20~30개 붙은 태그가 흔합니다. (→ 8장 컴포넌트화로 관리)
- 초기 학습 비용이 있습니다. 클래스 이름을 외우는 기간이 필요합니다.
- 클래스 이름을 동적으로 문자열 조합하면 동작하지 않습니다. (→ 9장, 실무에서 가장 자주 터지는 부분)
- 빌드 도구(Vite 등)가 필요합니다.
- 디자인 시스템이 없는 조직에서는 여전히 결과물이 제각각일 수 있습니다. Tailwind는 도구지 규율이 아닙니다.

> 백엔드 관점 비유: Tailwind는 Eloquent 같은 "고수준 추상화"가 아니라 오히려 **쿼리 빌더**에 가깝습니다. SQL을 감춰주는 게 아니라, SQL을 안전하고 일관되게 조립하도록 돕습니다. CSS 지식이 필요 없어지는 게 아니라, CSS 지식이 그대로 클래스 이름으로 매핑된다고 보는 편이 정확합니다.

---

## 2. 설치 (Laravel + Vite 중심)

### 2.1 사전 확인: 브라우저 지원 범위

v4는 최신 CSS 기능(`@property`, `color-mix()`, cascade layers)에 의존합니다.

- **Safari 16.4+, Chrome 111+, Firefox 128+**
- 그보다 낮은 버전을 지원해야 하는 프로젝트(공공/금융 등 구형 환경 대응이 필요한 경우)라면 **v3.4를 쓰는 것이 현실적**입니다. v3.4는 별도로 유지보수되고 있습니다.

이 문서는 v4 기준이며, v3와의 차이는 15장에 정리했습니다.

### 2.2 Laravel 프로젝트에 설치

```bash
npm install tailwindcss @tailwindcss/vite
```

`vite.config.js`:

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
    plugins: [
        laravel({
            input: ['resources/css/app.css', 'resources/js/app.js'],
            refresh: true,
        }),
        tailwindcss(),
    ],
});
```

`resources/css/app.css`:

```css
@import "tailwindcss";

/* Blade 템플릿과 페이지네이션 뷰까지 스캔 대상으로 지정 */
@source "../../vendor/laravel/framework/src/Illuminate/Pagination/resources/views/*.blade.php";
@source "../../storage/framework/views/*.php";
@source "../**/*.blade.php";
@source "../**/*.js";
```

레이아웃 Blade 파일:

```blade
<!doctype html>
<html lang="ko">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body class="bg-gray-50 text-gray-900">
    <h1 class="text-3xl font-bold underline">Hello</h1>
</body>
</html>
```

실행:

```bash
npm run dev     # 개발 (HMR)
npm run build   # 프로덕션 빌드
```

> **주의**: Laravel installer가 만든 프로젝트에 v3가 이미 들어있는 경우가 있습니다. `npm ls tailwindcss`로 버전을 확인하세요. v3 잔재(`tailwind.config.js`, `postcss.config.js`의 `tailwindcss`/`autoprefixer` 항목, `app.css`의 `@tailwind base;` 3줄)가 남아 있으면 충돌합니다. v4에서는 autoprefixer와 postcss-import가 내장이라 따로 필요 없습니다.

### 2.3 다른 환경

**순수 Vite (React/Vue 등)**

```bash
npm install tailwindcss @tailwindcss/vite
```
```js
// vite.config.js
import tailwindcss from '@tailwindcss/vite';
export default defineConfig({ plugins: [tailwindcss()] });
```
```css
/* src/index.css */
@import "tailwindcss";
```

**PostCSS 파이프라인 (Webpack, Next.js 등)**

```bash
npm install tailwindcss @tailwindcss/postcss postcss
```
```js
// postcss.config.mjs
export default { plugins: { "@tailwindcss/postcss": {} } };
```

**Webpack 전용 플러그인** (v4.2+): `@tailwindcss/webpack`도 있습니다.

**빌드 도구 없이 CLI만**

```bash
npm install tailwindcss @tailwindcss/cli
npx @tailwindcss/cli -i ./src/input.css -o ./dist/output.css --watch
```

레거시 PHP 프로젝트(Laravel이 아닌 경우)에 얹을 때 이 방식이 편합니다. 결과 CSS 파일만 배포에 포함시키면 됩니다.

**브라우저 빌드(`@tailwindcss/browser`)** 는 프로토타이핑 전용입니다. 런타임에 CSS를 생성하므로 프로덕션에는 쓰지 않습니다.

### 2.4 에디터 설정 (거의 필수)

- **Tailwind CSS IntelliSense** (VS Code / JetBrains 계열 모두 지원): 자동완성, 클래스에 마우스 올리면 실제 CSS 표시, 오타 경고. PhpStorm에는 내장 지원이 있습니다.
- **prettier-plugin-tailwindcss**: 클래스 순서를 자동 정렬 (13장 참고).

이 두 개 없이 시작하면 클래스 이름 외우는 데 불필요한 시간이 듭니다.

---

## 3. 클래스 이름 읽는 법 — CSS ↔ Tailwind 대응

### 3.1 핵심 규칙

대부분의 클래스는 `속성약어-값` 구조입니다.

| Tailwind | CSS | 비고 |
|---|---|---|
| `p-4` | `padding: 1rem` | p = padding |
| `px-4` | `padding-left/right: 1rem` | x = 가로축 |
| `py-2` | `padding-top/bottom: 0.5rem` | y = 세로축 |
| `pt-4` `pr-4` `pb-4` `pl-4` | 각 방향 padding | t/r/b/l |
| `m-4` `mx-auto` `-mt-2` | margin (음수는 앞에 `-`) | |
| `w-full` `h-screen` `size-10` | `width:100%` `height:100vh` `w+h:2.5rem` | |
| `text-lg` | `font-size: 1.125rem` + line-height | 폰트 크기 |
| `text-gray-700` | `color: ...` | 같은 `text-` 접두사가 크기와 색 둘 다 담당 |
| `font-semibold` | `font-weight: 600` | |
| `bg-blue-500` | `background-color` | |
| `border` `border-2` `border-t` | `border-width` | |
| `border-gray-200` | `border-color` | |
| `rounded-lg` | `border-radius` | |
| `flex` `grid` `block` `hidden` | `display` | |
| `items-center` | `align-items: center` | |
| `justify-between` | `justify-content: space-between` | |
| `gap-4` | `gap: 1rem` | |
| `shadow-md` | `box-shadow` | |
| `opacity-50` | `opacity: .5` | |
| `overflow-hidden` | `overflow` | |
| `absolute` `relative` `sticky` | `position` | |
| `top-0` `inset-0` `z-10` | 위치/z-index | |

처음에는 외우려 하기보다 **IntelliSense 자동완성으로 확인하면서 쓰다 보면** 자연스럽게 익혀집니다. 모르는 클래스는 공식 문서 검색(⌘K)이 가장 빠릅니다.

### 3.2 간격 스케일 (숫자의 의미)

기본 단위는 `--spacing: 0.25rem` (= 4px)입니다. 숫자에 이 값을 곱합니다.

```
p-1  → 0.25rem (4px)
p-2  → 0.5rem  (8px)
p-4  → 1rem    (16px)
p-6  → 1.5rem  (24px)
p-8  → 2rem    (32px)
p-12 → 3rem    (48px)
```

**숫자 × 4 = px** 로 암산하면 됩니다. v4에서는 정의되지 않은 숫자(`p-13`, `mt-27`)도 자동 계산되어 동작합니다.

특수 값: `p-px`(1px), `w-full`(100%), `w-1/2`(50%), `w-screen`(100vw), `h-dvh`(동적 뷰포트 높이, 모바일 주소창 대응).

### 3.3 색상 스케일

`색상이름-숫자` 형식이며 숫자는 **50(가장 밝음) ~ 950(가장 어두움)**입니다.

```
bg-gray-50   /* 거의 흰색, 페이지 배경용 */
bg-gray-100  /* 카드 배경, 구분선 */
bg-gray-500  /* 중간, 보조 텍스트 */
bg-gray-900  /* 본문 텍스트 */
```

기본 팔레트: `slate gray zinc neutral stone red orange amber yellow lime green emerald teal cyan sky blue indigo violet purple fuchsia pink rose`

**투명도 수식어**는 슬래시로 붙입니다.

```html
<div class="bg-black/50">   <!-- rgb 검정 50% 투명 -->
<div class="text-white/80">
<div class="border-blue-500/25">
```

v4의 기본 색상은 `oklch` 색 공간을 씁니다. 넓은 색역(P3) 디스플레이에서 더 선명하게 보이며, 밝기 단계가 지각적으로 균등합니다. 실무에서 신경 쓸 일은 거의 없지만, 디자이너가 준 hex와 미세하게 달라 보일 수 있다는 점만 알아두면 됩니다.

### 3.4 임의 값 (arbitrary values)

스케일에 없는 값이 꼭 필요할 때 대괄호를 씁니다.

```html
<div class="top-[117px] w-[calc(100%-2rem)] bg-[#1da1f2] text-[13px]">
<div class="grid-cols-[1fr_500px_2fr]">   <!-- 공백 대신 언더스코어 -->
```

CSS 변수를 참조할 때는 괄호를 씁니다.

```html
<div class="bg-(--brand-color)">   <!-- bg-[var(--brand-color)] 의 축약 -->
```

임의 속성(유틸리티가 아예 없는 속성)도 가능합니다.

```html
<div class="[mask-type:luminance] [--my-var:12px]">
```

> **주의**: 임의 값은 탈출구입니다. 남용하면 Tailwind를 쓰는 이유(디자인 토큰 일관성)가 사라집니다. 같은 임의 값을 두 번 이상 쓰게 되면 7장의 `@theme`에 토큰으로 등록하는 편이 낫습니다.

---

## 4. 레이아웃: Flex와 Grid

### 4.1 Flexbox

```html
<!-- 가로 정렬, 양끝 배치, 세로 가운데 -->
<div class="flex items-center justify-between gap-4">
  <span>왼쪽</span>
  <span>오른쪽</span>
</div>

<!-- 세로 스택 -->
<div class="flex flex-col gap-2">...</div>

<!-- 남는 공간 채우기 -->
<div class="flex">
  <aside class="w-64 shrink-0">사이드바</aside>
  <main class="flex-1 min-w-0">본문</main>
</div>
```

자주 쓰는 것:

| 클래스 | 의미 |
|---|---|
| `flex` / `inline-flex` | display |
| `flex-row` / `flex-col` | 주축 방향 (기본 row) |
| `flex-wrap` | 줄바꿈 허용 |
| `items-start/center/end/stretch/baseline` | 교차축 정렬 |
| `justify-start/center/end/between/around/evenly` | 주축 정렬 |
| `gap-4` `gap-x-2` `gap-y-6` | 아이템 간격 |
| `flex-1` | `flex: 1 1 0%` (남은 공간 차지) |
| `shrink-0` | 줄어들지 않음 |
| `grow` | 늘어남 |

> `min-w-0`은 flex 아이템 안에서 긴 텍스트가 넘칠 때 필요한 관용구입니다. flex 아이템의 기본 `min-width: auto` 때문에 `truncate`가 동작하지 않는 문제를 해결합니다.

### 4.2 Grid

```html
<!-- 3열 균등 -->
<div class="grid grid-cols-3 gap-6">...</div>

<!-- 반응형: 모바일 1열 → 태블릿 2열 → 데스크톱 4열 -->
<div class="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-4">...</div>

<!-- 아이템 병합 -->
<div class="grid grid-cols-12 gap-4">
  <div class="col-span-12 lg:col-span-8">본문</div>
  <div class="col-span-12 lg:col-span-4">사이드</div>
</div>

<!-- 자동 반응형 (미디어쿼리 없이) -->
<div class="grid grid-cols-[repeat(auto-fill,minmax(16rem,1fr))] gap-4">...</div>
```

### 4.3 컨테이너와 중앙 정렬

```html
<!-- 최대 폭 제한 + 가운데 + 좌우 여백 -->
<div class="mx-auto max-w-5xl px-4 sm:px-6 lg:px-8">
  ...
</div>
```

`container` 유틸리티도 있지만, 위 조합이 더 명시적이고 실무에서 흔히 쓰입니다.

### 4.4 컨테이너 쿼리 (v4 기본 내장)

뷰포트가 아니라 **부모 요소의 폭**을 기준으로 반응합니다. 사이드바 안이든 본문이든 재사용되는 카드 컴포넌트에 유용합니다.

```html
<div class="@container">
  <div class="flex flex-col @md:flex-row @md:items-center gap-4">
    <img class="w-full @md:w-32" src="...">
    <div>...</div>
  </div>
</div>
```

- `@sm:` `@md:` `@lg:` — 컨테이너 폭 기준
- `@max-md:` — 그 이하일 때
- 이름 지정: `@container/card` → `@md/card:flex-row`

---

## 5. Variant: 상태·반응형·다크모드

Variant는 **접두사 형태로 조건을 붙이는 문법**입니다. `조건:유틸리티` 구조이며, 여러 개를 겹칠 수 있습니다.

### 5.1 상태

```html
<button class="bg-blue-600 hover:bg-blue-700 active:bg-blue-800
               focus-visible:outline-2 focus-visible:outline-blue-500
               disabled:opacity-50 disabled:cursor-not-allowed">
  저장
</button>
```

주요 상태: `hover: focus: focus-visible: focus-within: active: visited: disabled: checked: required: invalid: placeholder: first: last: odd: even: empty:`

### 5.2 반응형 — 모바일 퍼스트가 기본

이 부분에서 처음에 헷갈리기 쉽습니다.

```html
<div class="text-sm md:text-base lg:text-lg">
```

- 접두사 없음 = **모든 화면** (모바일 포함)
- `md:` = **768px 이상**에서 적용 (= `@media (width >= 48rem)`)

즉 `md:`는 "태블릿에서만"이 아니라 "태블릿 이상 전부"입니다. 데스크톱 우선으로 생각하면 반대로 동작하는 것처럼 느껴집니다.

기본 브레이크포인트:

| 접두사 | 최소 폭 |
|---|---|
| `sm:` | 40rem (640px) |
| `md:` | 48rem (768px) |
| `lg:` | 64rem (1024px) |
| `xl:` | 80rem (1280px) |
| `2xl:` | 96rem (1536px) |

특정 구간만 지정하려면 `max-` 접두사를 조합합니다.

```html
<div class="md:max-lg:hidden">   <!-- 768px~1023px 구간에서만 숨김 -->
<div class="max-sm:text-center"> <!-- 640px 미만에서만 -->
```

### 5.3 부모/형제 상태에 반응 — `group`, `peer`

CSS만으로는 까다로운 패턴을 클래스로 처리합니다.

```html
<!-- 부모에 hover하면 자식 색 변경 -->
<a href="#" class="group flex items-center gap-3 p-4 hover:bg-gray-50">
  <svg class="size-5 text-gray-400 group-hover:text-blue-600">...</svg>
  <span class="text-gray-700 group-hover:text-gray-900">메뉴</span>
</a>

<!-- 형제 요소(체크박스) 상태에 반응 -->
<input type="checkbox" class="peer sr-only" id="toggle">
<label for="toggle" class="block p-4 border peer-checked:border-blue-500 peer-checked:bg-blue-50">
  선택 항목
</label>
```

이름을 붙여 중첩도 가능합니다: `group/item` → `group-hover/item:`

### 5.4 `has:`, `not:`, `data-*`, `aria-*`

```html
<!-- 자식에 체크된 input이 있으면 -->
<label class="border has-checked:border-blue-500 has-checked:bg-blue-50">...</label>

<!-- 특정 상태가 아닐 때 -->
<div class="not-first:border-t">...</div>

<!-- 데이터 속성 기반 (Alpine.js, Livewire와 궁합이 좋음) -->
<div data-state="open" class="hidden data-[state=open]:block">...</div>

<!-- ARIA 속성 기반 -->
<th aria-sort="ascending" class="aria-[sort=ascending]:bg-gray-100">...</th>
```

`data-*` variant는 Alpine.js와 조합하면 JS로 클래스를 토글하지 않고 상태 속성만 바꿔서 스타일을 제어할 수 있어 코드가 단순해집니다.

### 5.5 조합 순서

variant는 **왼쪽에서 오른쪽으로 중첩**됩니다.

```html
<div class="dark:md:hover:bg-gray-800">
```
읽는 순서: 다크모드 → 768px 이상 → hover.

순서가 다르면 생성되는 셀렉터도 달라질 수 있으므로, 팀에서는 Prettier 플러그인으로 순서를 통일하는 편이 좋습니다(13장).

---

## 6. 실전 컴포넌트 예제

### 6.1 버튼

```html
<button type="submit"
        class="inline-flex items-center justify-center gap-2
               rounded-md bg-blue-600 px-4 py-2
               text-sm font-medium text-white
               shadow-xs transition-colors
               hover:bg-blue-700
               focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-blue-600
               disabled:pointer-events-none disabled:opacity-50">
  저장
</button>
```

### 6.2 입력 폼

```html
<div class="space-y-1">
  <label for="email" class="block text-sm font-medium text-gray-700">
    이메일
  </label>
  <input type="email" id="email" name="email"
         class="block w-full rounded-md border border-gray-300 px-3 py-2
                text-sm placeholder:text-gray-400
                focus:border-blue-500 focus:ring-2 focus:ring-blue-500/20 focus:outline-none
                aria-invalid:border-red-500"
         @error('email') aria-invalid="true" @enderror>
  @error('email')
    <p class="text-sm text-red-600">{{ $message }}</p>
  @enderror
</div>
```

### 6.3 카드

```html
<article class="overflow-hidden rounded-lg border border-gray-200 bg-white shadow-sm
                transition hover:shadow-md">
  <img src="..." alt="" class="aspect-video w-full object-cover">
  <div class="p-5">
    <h3 class="text-lg font-semibold text-gray-900 line-clamp-2">제목</h3>
    <p class="mt-2 text-sm text-gray-600 line-clamp-3">요약 내용...</p>
    <div class="mt-4 flex items-center justify-between text-xs text-gray-500">
      <time datetime="2026-08-03">2026.08.03</time>
      <span>조회 1,204</span>
    </div>
  </div>
</article>
```

### 6.4 반응형 테이블 (모바일에서 카드로)

```html
<div class="overflow-x-auto">
  <table class="min-w-full divide-y divide-gray-200 text-sm">
    <thead class="bg-gray-50">
      <tr>
        <th class="px-4 py-3 text-left font-medium text-gray-600">이름</th>
        <th class="px-4 py-3 text-left font-medium text-gray-600">상태</th>
        <th class="px-4 py-3 text-right font-medium text-gray-600">금액</th>
      </tr>
    </thead>
    <tbody class="divide-y divide-gray-100">
      @foreach ($orders as $order)
      <tr class="hover:bg-gray-50">
        <td class="px-4 py-3">{{ $order->name }}</td>
        <td class="px-4 py-3">
          <span class="inline-flex rounded-full bg-green-100 px-2 py-0.5 text-xs font-medium text-green-800">
            완료
          </span>
        </td>
        <td class="px-4 py-3 text-right tabular-nums">{{ number_format($order->amount) }}</td>
      </tr>
      @endforeach
    </tbody>
  </table>
</div>
```

> `tabular-nums`는 숫자 폭을 고정해 금액 컬럼이 흔들리지 않게 합니다. `divide-y`는 자식 사이에만 구분선을 넣는 유틸리티입니다.

---

## 7. 설정: CSS-first 방식 (`@theme`, `@utility`)

v4의 가장 큰 변화입니다. **`tailwind.config.js`가 아니라 CSS 파일에서 설정**합니다.

### 7.1 `@theme` — 디자인 토큰 정의

```css
@import "tailwindcss";

@theme {
  /* 브랜드 색상 → bg-brand-500, text-brand-700 ... 자동 생성 */
  --color-brand-50:  oklch(0.97 0.02 250);
  --color-brand-500: oklch(0.62 0.19 250);
  --color-brand-700: oklch(0.48 0.18 250);

  /* 폰트 → font-sans, font-display */
  --font-sans: "Pretendard Variable", Pretendard, system-ui, sans-serif;
  --font-display: "Poppins", sans-serif;

  /* 브레이크포인트 추가 → 3xl:flex */
  --breakpoint-3xl: 120rem;

  /* 커스텀 그림자 → shadow-card */
  --shadow-card: 0 1px 3px rgb(0 0 0 / 0.08), 0 1px 2px rgb(0 0 0 / 0.04);

  /* 애니메이션 → animate-fade-in */
  --animate-fade-in: fade-in 0.3s ease-out;
}

@keyframes fade-in {
  from { opacity: 0; transform: translateY(4px); }
  to   { opacity: 1; transform: none; }
}
```

**핵심**: `@theme`에 정의한 변수는 (1) 유틸리티 클래스로 자동 생성되고, (2) 동시에 `:root`의 CSS 변수로도 노출됩니다. 따라서 일반 CSS에서 `var(--color-brand-500)`으로도 쓸 수 있습니다.

주요 네임스페이스:

| 변수 접두사 | 생성되는 유틸리티 |
|---|---|
| `--color-*` | `bg-*` `text-*` `border-*` `fill-*` 등 |
| `--font-*` | `font-*` |
| `--text-*` | `text-*` (크기) |
| `--spacing` | `p-*` `m-*` `gap-*` 등의 기준 단위 |
| `--breakpoint-*` | `sm:` `md:` 같은 반응형 접두사 |
| `--container-*` | `max-w-*`, 컨테이너 쿼리 `@md:` |
| `--radius-*` | `rounded-*` |
| `--shadow-*` | `shadow-*` |
| `--ease-*`, `--animate-*` | `ease-*`, `animate-*` |

기본값을 지우고 싶을 때:

```css
@theme {
  --color-*: initial;          /* 기본 팔레트 전부 제거 */
  --color-white: #fff;
  --color-brand-500: #0066cc;  /* 우리 팔레트만 사용 */
}
```

디자인 시스템을 엄격히 강제하려는 조직에서 유용합니다.

### 7.2 `@utility` — 커스텀 유틸리티

```css
/* 정적 유틸리티 */
@utility scrollbar-hidden {
  scrollbar-width: none;
  &::-webkit-scrollbar { display: none; }
}

/* 함수형 유틸리티: tab-2, tab-4 ... */
@utility tab-* {
  tab-size: --value(integer);
}
```

`@utility`로 만든 것은 `hover:scrollbar-hidden`처럼 variant와도 조합됩니다. 일반 CSS 클래스로 만들면 이게 안 됩니다.

### 7.3 `@custom-variant` — 커스텀 조건

```css
/* 다크모드를 class 기반으로 전환 (기본은 OS 설정 기반) */
@custom-variant dark (&:where(.dark, .dark *));

/* Livewire 로딩 상태 */
@custom-variant loading (&[wire\:loading]);
```

### 7.4 `@plugin`, `@source`, `@config`

```css
@import "tailwindcss";

/* 공식 플러그인 */
@plugin "@tailwindcss/forms";
@plugin "@tailwindcss/typography";

/* 스캔 경로 추가 (자동 감지가 못 잡는 경우) */
@source "../../packages/ui/src/**/*.blade.php";

/* 특정 경로 제외 */
@source not "../legacy";

/* 클래스명이 코드에 문자열로 안 나타날 때 강제 포함 (안전목록) */
@source inline("bg-red-100 bg-yellow-100 bg-green-100 text-red-700 text-yellow-700 text-green-700");

/* v3 스타일 JS 설정 파일을 계속 쓰고 싶을 때 */
@config "../../tailwind.config.js";
```

`@tailwindcss/typography`는 CMS/에디터로 들어온 HTML(기사 본문 등)에 `prose` 클래스 하나로 읽기 좋은 타이포그래피를 적용합니다. 게시판·기사 본문이 있는 서비스라면 사실상 필수입니다.

```blade
<div class="prose prose-lg max-w-none prose-headings:font-semibold">
  {!! $article->body !!}
</div>
```

### 7.5 `@apply`와 `@layer` — 쓸 곳과 안 쓸 곳

`@apply`는 유틸리티를 CSS 규칙 안으로 끌어옵니다.

```css
@layer components {
  .btn-primary {
    @apply inline-flex items-center rounded-md bg-blue-600 px-4 py-2 text-white;
    @apply hover:bg-blue-700;
  }
}
```

**동작은 하지만, 기본 전략으로 삼지 않는 편이 좋습니다.** 이유는 8장에서 다룹니다. 다음 경우에는 합리적입니다.

- 서드파티 라이브러리(에디터, 캘린더 등)가 생성하는 마크업을 스타일링할 때 — HTML을 우리가 못 건드리는 경우
- `body`, `a` 같은 기본 요소 스타일

```css
@layer base {
  body { @apply bg-gray-50 text-gray-900 antialiased; }
  a    { @apply text-blue-600 underline-offset-2 hover:underline; }
}
```

> Vue/Svelte의 `<style>` 블록이나 CSS Module에서 `@apply`를 쓰려면 해당 파일 상단에 `@reference "../app.css";`가 필요합니다. 그 블록은 메인 CSS와 별도로 컴파일되기 때문입니다.

---

## 8. 프로덕션 핵심 ①: 컴포넌트화 전략

Tailwind 도입 후 3개월쯤 지나면 반드시 마주치는 문제입니다.

```blade
{{-- 이 30개 클래스가 프로젝트 전체에 47번 복사되어 있다 --}}
<button class="inline-flex items-center justify-center gap-2 rounded-md bg-blue-600 px-4 py-2 text-sm font-medium text-white shadow-xs transition-colors hover:bg-blue-700 focus-visible:outline-2 ...">
```

버튼 색을 바꾸라는 요청이 오면 47곳을 고쳐야 합니다.

### 8.1 해법: `@apply`가 아니라 **템플릿 컴포넌트**

`@apply`로 `.btn-primary`를 만들면 결국 예전 CSS 프레임워크로 되돌아갑니다. 전역 클래스, 네이밍 고민, 삭제 못 하는 CSS가 다시 생깁니다. Tailwind 제작자도 이 방식을 권장하지 않습니다.

대신 **템플릿 엔진의 컴포넌트 기능**을 씁니다. 이건 이미 익숙한 도구입니다.

`resources/views/components/button.blade.php`:

```blade
@props([
    'variant' => 'primary',
    'size' => 'md',
    'type' => 'button',
])

@php
    $base = 'inline-flex items-center justify-center gap-2 rounded-md font-medium
             transition-colors focus-visible:outline-2 focus-visible:outline-offset-2
             disabled:pointer-events-none disabled:opacity-50';

    $variants = [
        'primary'   => 'bg-blue-600 text-white shadow-xs hover:bg-blue-700 focus-visible:outline-blue-600',
        'secondary' => 'bg-white text-gray-700 border border-gray-300 hover:bg-gray-50',
        'danger'    => 'bg-red-600 text-white hover:bg-red-700 focus-visible:outline-red-600',
        'ghost'     => 'text-gray-600 hover:bg-gray-100',
    ];

    $sizes = [
        'sm' => 'px-3 py-1.5 text-xs',
        'md' => 'px-4 py-2 text-sm',
        'lg' => 'px-5 py-2.5 text-base',
    ];

    $classes = trim("$base {$variants[$variant]} {$sizes[$size]}");
@endphp

<button type="{{ $type }}" {{ $attributes->class($classes) }}>
    {{ $slot }}
</button>
```

사용:

```blade
<x-button>저장</x-button>
<x-button variant="danger" size="sm" wire:click="delete">삭제</x-button>
<x-button variant="secondary" class="w-full">취소</x-button>
```

`$attributes->class(...)`는 컴포넌트 기본 클래스와 호출부에서 넘긴 클래스를 병합합니다.

### 8.2 조건부 클래스: Blade의 `@class`

```blade
<div @class([
    'rounded-md px-4 py-3 text-sm',
    'bg-green-50 text-green-800 border border-green-200' => $type === 'success',
    'bg-red-50 text-red-800 border border-red-200'       => $type === 'error',
    'opacity-60' => $dismissed,
])>
    {{ $message }}
</div>
```

`Illuminate\Support\Arr::toCssClasses()`가 내부에서 동작합니다. 조건이 true인 키만 포함됩니다.

### 8.3 React/Vue를 쓴다면

```jsx
import clsx from 'clsx';
import { twMerge } from 'tailwind-merge';

const cn = (...inputs) => twMerge(clsx(inputs));

function Button({ variant = 'primary', className, ...props }) {
  return (
    <button
      className={cn(
        'inline-flex items-center rounded-md px-4 py-2 text-sm font-medium',
        variant === 'primary' && 'bg-blue-600 text-white hover:bg-blue-700',
        variant === 'danger' && 'bg-red-600 text-white hover:bg-red-700',
        className,
      )}
      {...props}
    />
  );
}
```

`tailwind-merge`는 `px-4`와 `px-6`이 같이 들어오면 뒤엣것만 남깁니다. Tailwind는 CSS 우선순위가 클래스 작성 순서가 아니라 CSS 파일 내 순서로 결정되므로, 이 병합이 없으면 오버라이드가 의도대로 안 될 때가 있습니다. `class-variance-authority(cva)`를 얹으면 변형 관리가 더 체계적입니다.

Blade에서는 동일한 라이브러리가 표준화되어 있지 않으므로, 위 8.1처럼 **변형을 배열로 관리하고 오버라이드는 최소화**하는 쪽이 안전합니다.

### 8.4 언제 컴포넌트로 뽑을까

- 같은 클래스 조합이 **3번 이상** 반복되면
- 디자인 변경 요청이 여러 곳을 동시에 건드릴 것 같으면
- 반대로, 한 화면에만 있는 레이아웃은 굳이 뽑지 않습니다. 조기 추상화는 오히려 비용입니다.

---

## 9. 프로덕션 핵심 ②: 동적 클래스 함정

**실무에서 가장 자주, 그리고 배포 후에야 터지는 문제입니다.**

Tailwind는 소스 파일을 **정규식 기반 텍스트 스캔**으로 훑어 등장하는 클래스명만 CSS로 생성합니다. 코드를 실행하지도, 파싱해서 이해하지도 않습니다. 문자열을 조립해서 만든 클래스는 스캐너 눈에 보이지 않습니다.

```blade
{{-- 동작하지 않음 --}}
<div class="bg-{{ $color }}-500">
<div class="text-{{ $status === 'ok' ? 'green' : 'red' }}-600">
```

```jsx
// 동작하지 않음
<div className={`text-${size} bg-${color}-500`} />
```

개발 중에는 우연히 다른 곳에서 그 클래스를 써서 잘 되는 것처럼 보이다가, 그 코드를 지우는 순간 조용히 깨집니다.

### 해법 1: 전체 클래스명을 매핑 (권장)

```php
// app/Enums/OrderStatus.php 또는 헬퍼
$badgeClasses = match ($status) {
    'pending'   => 'bg-yellow-100 text-yellow-800 ring-yellow-600/20',
    'shipped'   => 'bg-blue-100 text-blue-800 ring-blue-600/20',
    'delivered' => 'bg-green-100 text-green-800 ring-green-600/20',
    'canceled'  => 'bg-gray-100 text-gray-800 ring-gray-600/20',
};
```

문자열 안에 **완전한 클래스명이 그대로 존재**하므로 스캐너가 인식합니다.

Enum에 넣어두면 관리가 깔끔합니다.

```php
enum OrderStatus: string
{
    case Pending = 'pending';
    case Shipped = 'shipped';

    public function badgeClasses(): string
    {
        return match ($this) {
            self::Pending => 'bg-yellow-100 text-yellow-800',
            self::Shipped => 'bg-blue-100 text-blue-800',
        };
    }
}
```

### 해법 2: CSS 변수로 우회 (값이 DB에서 오는 경우)

카테고리 색상처럼 값이 런타임에 결정되면 클래스로는 해결이 안 됩니다.

```blade
<div class="bg-(--cat-color) text-white"
     style="--cat-color: {{ $category->hex_color }}">
```

> 사용자 입력이 그대로 들어가는 경로라면 반드시 값 검증(hex 패턴 화이트리스트)을 하세요. CSS 값 주입도 공격 표면이 됩니다.

### 해법 3: 안전목록 (`@source inline`)

PHP 클래스 밖(예: DB에 저장된 설정값)에서 클래스명이 오는 경우:

```css
@source inline("bg-{red,yellow,green,blue}-{100,500,800}");
```

중괄호 확장을 지원합니다. 다만 CSS 크기가 늘어나므로 최소한으로 씁니다.

### 배포 파이프라인 관련 주의

- `resources/views` 외 경로(패키지, `app/View/Components`, PHP 클래스 안의 클래스 문자열 등)에 클래스명이 있으면 `@source`로 명시적으로 추가해야 합니다.
- **PHP 클래스 파일 안에 Tailwind 클래스 문자열을 쓴다면 그 경로도 스캔 대상**이어야 합니다. 위 Enum 예시가 그렇습니다.

```css
@source "../../app/**/*.php";
```

- `.gitignore`된 파일은 기본적으로 스캔에서 제외됩니다. 빌드 산출물이나 캐시 경로에 의존하지 마세요.

---

## 10. 다크모드 실전

### 10.1 기본 (OS 설정 추종)

```html
<div class="bg-white text-gray-900 dark:bg-gray-900 dark:text-gray-100">
```

별도 설정 없이 `prefers-color-scheme`을 따릅니다.

### 10.2 수동 토글 (사용자 선택 저장)

```css
/* app.css */
@custom-variant dark (&:where(.dark, .dark *));
```

```blade
{{-- <head> 안, CSS보다 먼저. 화면 깜빡임(FOUC) 방지용 인라인 스크립트 --}}
<script>
    (function () {
        const stored = localStorage.getItem('theme');
        const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
        if (stored === 'dark' || (!stored && prefersDark)) {
            document.documentElement.classList.add('dark');
        }
    })();
</script>
```

```js
// 토글
function setTheme(theme) { // 'light' | 'dark' | 'system'
    localStorage.setItem('theme', theme);
    const dark = theme === 'dark'
        || (theme === 'system' && matchMedia('(prefers-color-scheme: dark)').matches);
    document.documentElement.classList.toggle('dark', dark);
}
```

> 이 스크립트를 번들 파일이 아니라 인라인으로, `<head>` 상단에 두는 것이 중요합니다. 늦게 실행되면 밝은 화면이 한 프레임 보였다가 어두워집니다.

### 10.3 유지보수 관점: 시맨틱 토큰

`dark:` 접두사를 수백 군데 붙이는 대신, 의미 기반 토큰을 정의하면 관리가 쉬워집니다.

```css
@theme inline {
  --color-surface: var(--surface);
  --color-content: var(--content);
  --color-muted: var(--muted);
}

:root {
  --surface: oklch(1 0 0);
  --content: oklch(0.21 0.01 260);
  --muted:   oklch(0.55 0.01 260);
}

.dark {
  --surface: oklch(0.21 0.01 260);
  --content: oklch(0.98 0 0);
  --muted:   oklch(0.71 0.01 260);
}
```

```html
<div class="bg-surface text-content">
  <p class="text-muted">보조 설명</p>
</div>
```

`@theme inline`은 값을 그대로 인라인해서 다른 CSS 변수를 참조할 수 있게 합니다. 이 패턴을 쓰면 다크모드 대응이 **CSS 한 곳**에서 끝납니다. 규모가 있는 프로젝트라면 처음부터 이렇게 잡는 편이 낫습니다.

---

## 11. 접근성 체크리스트

Tailwind는 접근성을 자동으로 보장하지 않습니다. 다음은 프로덕션 릴리스 전에 확인할 항목입니다.

**1. 포커스 표시를 지우지 말 것**

```html
<!-- 지양: 키보드 사용자가 현재 위치를 알 수 없음 -->
<button class="outline-none">

<!-- 권장: 마우스 클릭 시에는 안 보이고, 키보드 탐색 시에만 표시 -->
<button class="focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-blue-600">
```

v4에서 `outline-none`은 `outline-hidden`으로 바뀌었습니다. `outline-hidden`은 Windows 고대비 모드에서는 윤곽선을 유지합니다.

**2. 스크린리더 전용 텍스트**

```html
<button class="p-2">
  <svg class="size-5" aria-hidden="true">...</svg>
  <span class="sr-only">메뉴 열기</span>
</button>
```

`sr-only`는 시각적으로만 숨기고 스크린리더에는 읽힙니다. `hidden`은 스크린리더에서도 사라지므로 다릅니다.

**3. 색상 대비**

`text-gray-400`을 흰 배경 본문에 쓰면 WCAG AA(4.5:1)를 통과하지 못합니다. 본문은 `text-gray-600` 이상, 큰 제목은 `text-gray-500` 정도가 안전선입니다. 색상만으로 상태를 구분하지 말고 아이콘이나 텍스트를 병행하세요.

**4. 모션 민감성**

```html
<div class="motion-safe:animate-fade-in motion-reduce:transition-none">
```

**5. 터치 타깃 크기**

모바일에서 최소 44×44px 권장 → `min-h-11 min-w-11` (11 × 4px = 44px).

**6. 폼 플러그인**

```css
@plugin "@tailwindcss/forms";
```

브라우저별로 제각각인 기본 폼 요소 스타일을 표준화합니다. 접근성 기본값도 함께 정리됩니다.

---

## 12. 빌드·배포·성능

### 12.1 빌드 흐름

```bash
npm run dev     # 개발: HMR, 요청된 클래스를 즉시 생성
npm run build   # 배포: 사용된 클래스만 포함한 최소 CSS + 해시 파일명
```

Laravel의 `@vite()` 디렉티브는 개발 모드에서는 dev 서버를, 프로덕션에서는 `public/build/manifest.json`을 읽어 해시된 파일을 참조합니다. 캐시 무효화가 자동 처리됩니다.

### 12.2 배포 체크리스트

```bash
# CI/CD 또는 배포 스크립트
npm ci
npm run build        # ← 이 단계를 빠뜨리면 스타일 없는 페이지가 배포됩니다
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

- `public/build/`를 저장소에 커밋할지 여부를 팀 규칙으로 정하세요. 커밋하지 않는 쪽이 일반적이며, 그 경우 **배포 서버에 Node가 있거나 CI에서 빌드 후 산출물을 전송**해야 합니다.
- `php artisan view:cache`를 쓴다면 `storage/framework/views/*.php`도 `@source`에 넣어두는 것이 안전합니다(2.2 예제에 포함되어 있습니다).
- 무중단 배포 시 이전 버전 사용자가 구 CSS를 참조할 수 있으므로, 이전 빌드 산출물을 일정 기간 남겨두는 전략을 고려하세요.

### 12.3 성능 관련 사실

- 최종 CSS는 대부분 **10~50KB(gzip 후 그보다 작음)** 수준입니다. 페이지 수가 늘어도 클래스 종류가 늘지 않으면 크기가 거의 증가하지 않습니다. 이게 전통적 CSS와 가장 큰 차이입니다.
- v4의 엔진(Oxide, Rust 기반)은 전체 빌드가 수십 ms, 증분 빌드는 마이크로초 단위입니다.
- CSS는 렌더링 차단 리소스이므로 `<head>`에 두는 게 맞습니다. Tailwind 결과물은 작아서 대부분의 프로젝트에서 critical CSS 분리는 불필요합니다.
- 웹폰트가 CSS보다 큰 경우가 많습니다. 성능 최적화는 폰트 서브셋과 `font-display: swap`부터 보는 편이 효과적입니다.

### 12.4 하지 말아야 할 것

- 브라우저 런타임 빌드(`@tailwindcss/browser`)를 프로덕션에 사용
- 빌드 없이 `node_modules`의 CSS를 직접 링크
- 생성된 CSS 파일을 손으로 수정

---

## 13. 팀 협업과 유지보수

### 13.1 클래스 순서 자동 정렬

```bash
npm install -D prettier prettier-plugin-tailwindcss
```

```json
// .prettierrc
{
  "plugins": ["prettier-plugin-tailwindcss"],
  "tailwindStylesheet": "./resources/css/app.css"
}
```

Blade 파일까지 포맷하려면 `prettier-plugin-blade`를 함께 씁니다. 클래스 순서가 통일되면 코드 리뷰에서 diff 노이즈가 크게 줄어듭니다. 사람이 순서를 신경 쓸 필요가 없어지는 게 실질적인 이득입니다.

### 13.2 린팅

v4의 IntelliSense 확장은 오타, 존재하지 않는 클래스, 상충하는 클래스(`p-2 p-4` 동시 사용) 등을 경고합니다. CI에서 강제하려면 `eslint-plugin-tailwindcss`(JS 프로젝트) 또는 커스텀 스크립트를 검토하세요.

### 13.3 팀 규칙으로 정해두면 좋은 것

| 항목 | 권장 |
|---|---|
| 임의 값 `[...]` 사용 | 리뷰에서 근거 확인. 반복되면 `@theme` 토큰으로 승격 |
| `@apply` 사용 | 서드파티 마크업 대응 등 예외 상황으로 제한 |
| 컴포넌트 추출 기준 | 동일 조합 3회 반복 시 |
| 색상 | `@theme`에 정의된 시맨틱 토큰만 사용. 원시 팔레트 직접 사용 최소화 |
| 클래스 줄바꿈 | 80~100자 넘으면 논리 단위(레이아웃 / 색상 / 상태)로 줄 나눔 |
| 반응형 | 모바일 기준으로 먼저 작성 후 `md:` `lg:` 추가 |

### 13.4 긴 클래스 목록 정리 방법

```blade
<button class="inline-flex items-center justify-center gap-2
               rounded-md px-4 py-2
               text-sm font-medium text-white
               bg-blue-600 shadow-xs
               transition-colors hover:bg-blue-700
               focus-visible:outline-2 focus-visible:outline-blue-600">
```

HTML은 클래스 속성 내 개행을 공백으로 처리하므로 안전합니다. **레이아웃 → 크기/간격 → 타이포 → 색상 → 상태** 순으로 묶으면 읽기 좋습니다.

---

## 14. 한국어 사이트에서 자주 필요한 것들

### 14.1 줄바꿈 — `break-keep`

한국어는 기본 `word-break: normal`에서 어절 중간이 끊길 수 있습니다.

```html
<p class="break-keep">
  긴 한국어 문장이 어색하게 끊기지 않고 어절 단위로 줄바꿈됩니다.
</p>
```

제목이나 버튼 라벨에 특히 체감이 큽니다. `text-pretty`(줄 끝 고아 단어 방지), `text-balance`(제목 줄 길이 균등화)와 함께 쓰면 좋습니다.

```html
<h1 class="text-3xl font-bold text-balance break-keep">기사 제목이 들어갑니다</h1>
```

### 14.2 폰트

```css
@theme {
  --font-sans: "Pretendard Variable", Pretendard, -apple-system,
               BlinkMacSystemFont, system-ui, "Malgun Gothic", sans-serif;
}
```

한글 웹폰트는 용량이 크므로(전체 셋 수 MB) 서브셋 폰트나 `woff2` + `unicode-range` 분할, CDN 사용을 검토하세요.

### 14.3 숫자·금액 표기

```html
<span class="tabular-nums">1,234,567원</span>
```

### 14.4 논리 속성 (다국어 대응)

`ml-4` 대신 `ms-4`(margin-inline-start)를 쓰면 RTL 언어에서도 자동으로 방향이 뒤집힙니다. 다국어 계획이 있다면 처음부터 논리 속성 유틸리티를 쓰는 편이 낫습니다.

---

## 15. v3 → v4 차이 (인터넷 예제 볼 때 주의)

검색으로 나오는 예제 상당수가 아직 v3 기준입니다. 그대로 붙여넣으면 동작하지 않거나 다르게 보입니다.

### 15.1 구조적 변화

| 항목 | v3 | v4 |
|---|---|---|
| 진입점 | `@tailwind base; @tailwind components; @tailwind utilities;` | `@import "tailwindcss";` |
| 설정 | `tailwind.config.js` | CSS의 `@theme` (JS 설정은 `@config`로 계속 사용 가능) |
| 스캔 경로 | `content: [...]` 배열 | 자동 감지 + `@source` |
| 플러그인 | config의 `plugins: []` | `@plugin "..."` |
| PostCSS | `tailwindcss`, `autoprefixer`, `postcss-import` 필요 | `@tailwindcss/postcss` 하나 (나머지 내장) |
| 다크모드 class 전략 | `darkMode: 'class'` | `@custom-variant dark (...)` |

### 15.2 이름이 바뀐 유틸리티

| v3 | v4 |
|---|---|
| `shadow-sm` | `shadow-xs` |
| `shadow` | `shadow-sm` |
| `rounded-sm` | `rounded-xs` |
| `rounded` | `rounded-sm` |
| `blur`, `drop-shadow` 등 무접미사 | 각각 `-sm`이 붙는 형태로 이동 |
| `outline-none` | `outline-hidden` (진짜 `outline: none`은 `outline-none`) |
| `ring` (3px) | `ring` = 1px, v3와 같게 하려면 `ring-3` |
| `bg-opacity-50` | `bg-black/50` |
| `flex-shrink-0` / `flex-grow` | `shrink-0` / `grow` |

### 15.3 기본값 변화

- **테두리 색**: v3 `gray-200` → v4 `currentColor`. `<div class="border">`만 쓰면 글자색과 같은 테두리가 나옵니다. 색을 명시하거나 base 레이어에서 기본값을 지정하세요.

```css
@layer base {
  *, ::after, ::before { border-color: var(--color-gray-200); }
}
```

- **`ring` 기본**: 3px 파란색 → 1px currentColor
- **placeholder 색**: `gray-400` → 현재 글자색의 50% 투명도

### 15.4 마이그레이션 도구

```bash
npx @tailwindcss/upgrade
```

v3 프로젝트를 자동 변환합니다. 별도 브랜치에서 실행하고 diff를 검토하세요. 커스텀 플러그인이나 복잡한 config는 수동 확인이 필요합니다.

---

## 16. 자주 하는 실수 모음

1. **동적 클래스 문자열 조합** (`bg-{{ $color }}-500`) → 9장. 가장 자주 터집니다.
2. **`@apply` 남용** → 결국 예전 CSS 프레임워크로 회귀. 8장 참고.
3. **임의 값 `[13px]` 남발** → 디자인 토큰이 무의미해집니다. 두 번 이상 쓰면 `@theme`으로.
4. **반응형을 데스크톱 기준으로 생각** → `md:`는 "768px 이상 전부"입니다.
5. **`outline-none`으로 포커스 링 제거** → 접근성 문제. `focus-visible:` 사용.
6. **`space-x-4`와 flex `gap-4` 혼용** → 새 코드에서는 `gap`을 기본으로. `space-*`는 자식 선택자를 쓰기 때문에 조건부 렌더링과 상성이 나쁩니다.
7. **v3 예제 복붙** → 15장의 이름 변경 확인.
8. **배포 시 `npm run build` 누락** → 스타일 없는 페이지 배포.
9. **다크모드 초기화 스크립트를 번들에 넣음** → 첫 화면 깜빡임.
10. **컴포넌트 추출 시점을 놓침** → 같은 클래스 덩어리가 수십 곳에 복사됨.
11. **`min-w-0` 누락** → flex 안에서 `truncate`가 동작하지 않음.
12. **PHP 클래스 파일을 `@source`에 포함하지 않음** → Enum/헬퍼에 넣은 클래스가 빌드에서 누락.

---

## 17. 치트시트

### 간격·크기
```
p-4 px-6 py-2 pt-3   m-4 mx-auto -mt-2
w-full w-1/2 w-64 max-w-md min-w-0
h-screen h-dvh size-10  aspect-video aspect-square
space-y-4  gap-4 gap-x-2
```

### 타이포그래피
```
text-xs sm base lg xl 2xl ... 9xl
font-light normal medium semibold bold
text-left center right   leading-tight relaxed   tracking-tight wide
truncate  line-clamp-2  break-keep  text-balance  text-pretty
uppercase  tabular-nums  antialiased
```

### 색상
```
text-gray-700  bg-blue-600  border-red-300
bg-black/50  text-white/80
bg-gradient-to-r from-blue-500 to-purple-600
divide-y divide-gray-200
```

### 레이아웃
```
flex inline-flex grid block inline-block hidden
flex-col flex-wrap flex-1 shrink-0 grow
items-center justify-between
grid-cols-3 col-span-2 auto-rows-min
absolute relative fixed sticky  inset-0 top-4 z-50
overflow-hidden overflow-x-auto
```

### 테두리·그림자
```
border border-2 border-t  rounded rounded-lg rounded-full
shadow-xs shadow-sm shadow-md shadow-lg  ring-2 ring-blue-500
outline-2 outline-offset-2
```

### 상태·반응형
```
hover: focus: focus-visible: active: disabled: checked:
group-hover: peer-checked: has-checked: not-first:
sm: md: lg: xl: 2xl:   max-md:   dark:
data-[state=open]:  aria-[expanded=true]:
@container  @md:  @max-sm:
```

### 전환·애니메이션
```
transition transition-colors duration-200 ease-out delay-100
animate-spin animate-pulse animate-bounce
motion-safe: motion-reduce:
scale-105 rotate-3 translate-x-2
```

### 기타
```
cursor-pointer select-none pointer-events-none
sr-only  scroll-smooth  isolate  will-change-transform
```

---

## 18. 학습 경로와 참고 자료

### 순서 제안

1. **1~3장**을 읽고 개념과 클래스 규칙만 잡습니다.
2. 기존 프로젝트의 **작은 페이지 하나**를 Tailwind로 다시 작성해 봅니다. 새 프로젝트보다 기존 화면 이식이 학습 효과가 큽니다.
3. 손에 익으면 **8·9장**을 다시 읽습니다. 이 두 장이 프로덕션 품질을 좌우합니다.
4. `@theme`로 디자인 토큰을 잡고(7장), 다크모드가 필요하면 10장의 시맨틱 토큰 방식으로 설계합니다.
5. 배포 전 **11·12장** 체크리스트를 돌립니다.

### 참고 링크

- 공식 문서: <https://tailwindcss.com/docs> — 검색(⌘K)이 잘 되어 있어 사전처럼 씁니다.
- v4 업그레이드 가이드: <https://tailwindcss.com/docs/upgrade-guide>
- Laravel 설치 가이드: <https://tailwindcss.com/docs/installation/framework-guides/laravel/vite>
- Tailwind Plus (유료 컴포넌트): <https://tailwindcss.com/plus> — 마크업 참고용으로 봐도 학습에 도움이 됩니다.
- Headless UI (접근성 처리된 무스타일 컴포넌트, React/Vue): <https://headlessui.com>
- Heroicons: <https://heroicons.com>
- Flowbite / daisyUI: 무료 컴포넌트 라이브러리. 초기 속도는 빠르지만 프로젝트 디자인 시스템과 충돌할 수 있으니 도입 전 검토가 필요합니다.

### 실무에서 함께 쓰면 좋은 것

| 도구 | 용도 |
|---|---|
| `@tailwindcss/forms` | 폼 요소 기본 스타일 정규화 |
| `@tailwindcss/typography` | 기사 본문 등 CMS HTML에 `prose` 적용 |
| `prettier-plugin-tailwindcss` | 클래스 순서 자동 정렬 |
| Tailwind CSS IntelliSense | 자동완성·검증 |
| Alpine.js | Blade 환경에서 가벼운 상호작용. `data-*` variant와 궁합이 좋음 |
| Livewire | 서버 렌더링 유지하면서 동적 UI. `wire:loading` 등을 커스텀 variant로 연결 가능 |

---

*이 문서는 Tailwind CSS v4.3 기준으로 작성되었습니다. Tailwind는 마이너 릴리스가 잦으므로(대략 월 1회 내외), 새 유틸리티나 변경 사항은 공식 릴리스 노트를 주기적으로 확인하는 편이 좋습니다.*

---
title: Tailwind Variants
category: frontend
tags: [frontend, tailwind, css, responsive, dark-mode]
related: [[Tailwind CSS]], [[Tailwind Layout]], [[Tailwind Dark Mode]]
---

# Tailwind Variants

`조건:유틸리티` 형태로 상태·반응형·다크모드 등 조건을 클래스에 접두사로 붙이는 문법. 여러 개를 겹칠 수 있다.

## 핵심 개념

### 상태

```html
<button class="bg-blue-600 hover:bg-blue-700 active:bg-blue-800
               focus-visible:outline-2 focus-visible:outline-blue-500
               disabled:opacity-50 disabled:cursor-not-allowed">
  저장
</button>
```

주요 상태: `hover: focus: focus-visible: focus-within: active: visited: disabled: checked: required: invalid: placeholder: first: last: odd: even: empty:`

### 반응형 — 모바일 퍼스트가 기본

접두사 없음 = **모든 화면**(모바일 포함). `md:`는 "태블릿에서만"이 아니라 **"768px 이상 전부"**다. 데스크톱 우선으로 생각하면 반대로 동작하는 것처럼 느껴지는 지점이라 헷갈리기 쉽다.

| 접두사 | 최소 폭 |
|---|---|
| `sm:` | 640px |
| `md:` | 768px |
| `lg:` | 1024px |
| `xl:` | 1280px |
| `2xl:` | 1536px |

특정 구간만 지정하려면 `max-`를 조합한다: `md:max-lg:hidden`(768~1023px에서만 숨김), `max-sm:text-center`(640px 미만).

### 부모/형제 상태 — `group`, `peer`

```html
<a href="#" class="group flex items-center gap-3 p-4 hover:bg-gray-50">
  <svg class="size-5 text-gray-400 group-hover:text-blue-600">...</svg>
  <span class="text-gray-700 group-hover:text-gray-900">메뉴</span>
</a>

<input type="checkbox" class="peer sr-only" id="toggle">
<label for="toggle" class="block p-4 border peer-checked:border-blue-500 peer-checked:bg-blue-50">
  선택 항목
</label>
```

이름을 붙여 중첩 가능: `group/item` → `group-hover/item:`.

### `has:`, `not:`, `data-*`, `aria-*`

```html
<label class="border has-checked:border-blue-500 has-checked:bg-blue-50">...</label>
<div class="not-first:border-t">...</div>
<div data-state="open" class="hidden data-[state=open]:block">...</div>
<th aria-sort="ascending" class="aria-[sort=ascending]:bg-gray-100">...</th>
```

`data-*` variant는 Alpine.js와 조합하면 JS로 클래스를 토글하지 않고 상태 속성만 바꿔서 스타일을 제어할 수 있어 코드가 단순해진다.

### 조합 순서

variant는 **왼쪽에서 오른쪽으로 중첩**된다. `dark:md:hover:bg-gray-800`은 다크모드 → 768px 이상 → hover 순으로 읽는다. 순서가 다르면 생성되는 셀렉터도 달라질 수 있으므로 팀에서는 Prettier 플러그인(`prettier-plugin-tailwindcss`)으로 순서를 통일하는 편이 좋다.

## 참고

- [[Tailwind Layout]] — 반응형 그리드/컨테이너 쿼리
- [[Tailwind Dark Mode]] — `dark:` variant를 활용한 다크모드 구현
- [[Tailwind CSS]] — 개요
- 소스: `2026-08-03_Tailwind CSS 실전 가이드.md`

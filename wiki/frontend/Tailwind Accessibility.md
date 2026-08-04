---
title: Tailwind Accessibility
category: frontend
tags: [frontend, tailwind, accessibility, a11y]
related: [[Tailwind CSS]], [[Tailwind Variants]], [[Form Request]]
---

# Tailwind Accessibility

Tailwind는 접근성을 자동으로 보장하지 않는다. 프로덕션 릴리스 전에 확인할 체크리스트.

## 핵심 개념

**1. 포커스 표시를 지우지 말 것**

```html
<!-- 지양: 키보드 사용자가 현재 위치를 알 수 없음 -->
<button class="outline-none">

<!-- 권장: 마우스 클릭 시에는 안 보이고, 키보드 탐색 시에만 표시 -->
<button class="focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-blue-600">
```

v4에서 `outline-none`은 `outline-hidden`으로 이름이 바뀌었다 ([[Tailwind v3 to v4 Migration]]). `outline-hidden`은 Windows 고대비 모드에서는 윤곽선을 유지한다.

**2. 스크린리더 전용 텍스트**

```html
<button class="p-2">
  <svg class="size-5" aria-hidden="true">...</svg>
  <span class="sr-only">메뉴 열기</span>
</button>
```

`sr-only`는 시각적으로만 숨기고 스크린리더에는 읽힌다. `hidden`은 스크린리더에서도 사라지므로 다르다.

**3. 색상 대비**

`text-gray-400`을 흰 배경 본문에 쓰면 WCAG AA(4.5:1)를 통과하지 못한다. 본문은 `text-gray-600` 이상, 큰 제목은 `text-gray-500` 정도가 안전선이다. 색상만으로 상태를 구분하지 말고 아이콘이나 텍스트를 병행한다.

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

브라우저별로 제각각인 기본 폼 요소 스타일을 표준화한다. 접근성 기본값도 함께 정리된다.

## Laravel 구현

```blade
<div class="space-y-1">
  <label for="email" class="block text-sm font-medium text-gray-700">이메일</label>
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

`aria-invalid` 상태를 [[Form Request]]의 `@error` 디렉티브와 연결하면 서버 검증 실패가 시각적 표시와 스크린리더 안내 양쪽에 자동 반영된다.

## 주의사항 / 안티패턴

- `outline-none`(v3 기준, v4는 `outline-hidden`)으로 포커스 링을 제거하는 것이 가장 흔한 접근성 실수다. 반드시 `focus-visible:`로 대체할 것.
- `hidden`과 `sr-only`를 혼동하면 스크린리더 사용자에게 필요한 정보가 아예 빠질 수 있다.

## 참고

- [[Tailwind Variants]] — `focus-visible:`, `motion-safe:` 등 상태 variant
- [[Tailwind v3 to v4 Migration]] — `outline-none` → `outline-hidden` 이름 변경
- [[Form Request]] — 서버 검증과 `aria-invalid` 연동
- 소스: `2026-08-03_Tailwind CSS 실전 가이드.md`

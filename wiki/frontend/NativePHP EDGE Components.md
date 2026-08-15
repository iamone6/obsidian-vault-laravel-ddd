---
title: NativePHP EDGE Components
category: frontend
tags: [frontend, nativephp, mobile, components]
related: [[NativePHP SuperNative Architecture]], [[NativePHP Mobile Routing and Navigation]], [[Blade Component Basics]]
---

# NativePHP EDGE Components

Blade에서 `<native:*>` 형태로 사용하는 네이티브 UI 컴포넌트 전체 목록. 실제 화면을 짤 때 참고용으로 쓴다.

## 핵심 개념

각 태그는 [[NativePHP SuperNative Architecture]]에서 설명한 대로 스타일링된 HTML 엘리먼트가 아니라 그 플랫폼의 진짜 네이티브 위젯(SwiftUI/Jetpack Compose)으로 매핑된다.

### 레이아웃 / 구조

`column` · `row` · `stack` · `spacer` · `divider` · `scroll-view` · `layout`(Layout & Styling) · `safe-area` · `positioning`

### 입력

`button` · `button-group` · `text-input` · `checkbox` · `toggle` · `radio-group` · `select` · `slider` · `pressable` · `gesture-area`

### 표시

`text` · `icon` · `image` · `badge` · `chip` · `progress-bar` · `activity-indicator` · `shapes` · `canvas`

### 목록 / 컬렉션

`list` · `virtual-list` · `lazy-grid` · `carousel` · `accordion` · `refreshable`

### 내비게이션 / 오버레이

`top-bar` · `bottom-nav` · `side-nav` · `tab-row` · `modal` · `bottom-sheet` · `menus` · `fab`(Floating Action Button)

### 기타

`web-view` — 웹뷰를 컴포넌트로 삽입. [[NativePHP SuperNative Architecture]]의 마이그레이션 전략에서 쓰는 그 태그다.

## Laravel 구현

내비게이션 관련 컴포넌트(`pressable`, `bottom-nav-item` 등)는 [[NativePHP Mobile Routing and Navigation]]의 `@navigate` 디렉티브와 함께 쓰인다.

```blade
<native:pressable @navigate="/item/42">
    <native:text>항목 보기</native:text>
</native:pressable>
```

Blade 컴포넌트 문법(`@props`, `$slot` 등, [[Blade Component Basics]])과 태그 작성 감각은 비슷하지만, EDGE 컴포넌트는 자체 Blade 파일이 아니라 프레임워크가 제공하는 빌트인 네이티브 위젯이라는 점이 다르다.

## 주의사항 / 안티패턴

- 목록이 크면 `list` 대신 `virtual-list`를 검토한다 — 가상화 없이 큰 목록을 `list`나 반복 렌더링으로 그리면 저사양 기기에서 스크롤 성능이 떨어질 수 있다.
- `web-view` 컴포넌트를 EDGE 컴포넌트들과 섞어 쓸 때는 웹뷰 안쪽은 SuperNative의 공유 메모리 모델 밖이라는 점을 기억한다 — 웹뷰-네이티브 간 통신은 별도 브리지를 거친다.

## 참고

- [[NativePHP SuperNative Architecture]] — EDGE 컴포넌트가 네이티브 위젯으로 매핑되는 원리
- [[NativePHP Mobile Routing and Navigation]] — `pressable` 등과 `@navigate` 조합
- [[Blade Component Basics]] — Blade 컴포넌트 문법과의 감각 비교
- 소스: `2026-08-12_NativePHP for Mobile 실무 가이드.md`

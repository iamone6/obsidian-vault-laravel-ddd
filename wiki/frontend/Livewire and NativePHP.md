---
title: Livewire and NativePHP
category: frontend
tags: [frontend, livewire, nativephp, mobile]
related: [[Livewire Overview]], [[Livewire Alpine Integration]], [[Livewire Advanced v4 Features]], [[Livewire URL and Navigation]], [[NativePHP Mobile Overview]], [[NativePHP SuperNative Architecture]]
---

# Livewire and NativePHP

NativePHP 모바일 앱 맥락에서 Livewire를 배워야 하는 이유와, 학습 우선순위·모바일 전용 고려사항.

## 핵심 개념

### 현재 상황 (2026년 8월 기준)

- **NativePHP for Mobile v3** (stable): 웹뷰 기반. Blade/Livewire/React/Vue 모두 사용 가능. Livewire 4의 ⚡ 이모지 파일명 처리 이슈도 해결됨.
- **NativePHP for Mobile v4** (beta): **SuperNative** 아키텍처 도입. 웹뷰를 제거하고 **EDGE 컴포넌트**(Blade 태그 → SwiftUI/Jetpack Compose 네이티브 뷰)로 렌더링. 웹뷰는 여전히 하나의 컴포넌트로 사용 가능해서 화면 단위로 점진 이전할 수 있음.

> v4의 EDGE는 "Livewire처럼 느껴지도록" 설계됐다. `Route::native()`로 화면을 등록하고 PHP 컴포넌트 클래스가 화면을 구동하는 방식이다. **즉, Livewire를 배워두면 v3 웹뷰 방식에도, v4 EDGE 방식에도 그대로 사고방식이 이어진다.** 시간 투자가 헛되지 않는다. NativePHP 자체의 상세한 설치·아키텍처·라우팅은 [[NativePHP Mobile Overview]]와 [[NativePHP SuperNative Architecture]], [[NativePHP Mobile Routing and Navigation]]에 별도로 정리되어 있다.

또한 [[Livewire Overview]]에서 다뤘듯, Livewire의 최대 약점인 "네트워크 왕복"이 NativePHP에서는 localhost 호출이 되어 지연이 사실상 0이다 — NativePHP에서 Livewire를 권장하는 핵심 이유다. [[NativePHP SuperNative Architecture]]에 따르면 실제로는 이보다 한 단계 더 나아가, v4 SuperNative는 PHP와 네이티브 레이어가 **메모리를 직접 공유**해서 네트워크 계층 자체가 존재하지 않는다.

## Laravel 구현

### 학습 우선순위

**필수 (여기까지가 90%)**
- [[Livewire Render Cycle]] — 특히 "상태가 왕복한다"는 감각
- [[Livewire Installation and Components]]
- [[Livewire Properties]] + `wire:model`
- [[Livewire Actions]]
- [[Livewire Forms and Validation]]

**바로 뒤따라 필요한 것**
- [[Livewire Lifecycle Hooks]] (`mount`, `updated`)
- [[Livewire Computed Properties]]
- [[Livewire Rendering and wire-key]] — 리스트를 만들면 무조건 만남
- [[Livewire Loading States]]

**모바일에서 특히 유용**
- [[Livewire Events]] — 네이티브 이벤트(카메라 결과, 생체 인증 결과 등)를 Livewire 컴포넌트로 받을 때. 실제 사용 가능한 네이티브 API 목록은 [[NativePHP Core Plugins]] 참고
- [[Livewire Alpine Integration]] — 터치 반응처럼 즉각적이어야 하는 것들
- [[Livewire Advanced v4 Features]]의 Islands/Lazy — 저사양 기기에서 초기 렌더링 부담을 줄일 때

**당장은 우선순위 낮음**
- [[Livewire Pagination and File Uploads]]의 페이지네이션 (모바일에서는 무한 스크롤이 더 흔함)
- [[Livewire URL and Navigation]]의 URL 쿼리 (모바일 앱에 URL 개념이 약함)
- `wire:navigate` (v3 웹뷰에서는 유용, v4 EDGE에서는 네이티브 내비게이션 사용)

### 모바일 환경에서 추가로 신경 쓸 것

1. **snapshot 크기**: 저사양 기기에서는 JSON 직렬화/파싱 비용이 데스크톱보다 체감된다. public 프로퍼티를 최소화한다.
2. **로컬 SQLite**: 기기 안에서 도는 앱이므로 DB도 로컬이다. N+1 쿼리가 네트워크 없이도 느려질 수 있다.
3. **오프라인 우선**: 서버 API를 호출하는 부분은 실패를 전제로 설계해야 한다. `wire:offline`과 큐를 조합한다.
4. **터치 지연**: 버튼 탭 → 서버 왕복 → 화면 갱신 사이의 수십 ms도 모바일에서는 느껴진다. 가능한 것은 Alpine으로 낙관적 UI(optimistic UI)를 먼저 그리고, 서버 응답으로 확정하는 패턴을 쓴다.

```blade
{{-- 낙관적 UI 예시 --}}
<div x-data="{ done: @js($todo->completed) }">
    <input type="checkbox" x-model="done" @change="$wire.toggle({{ $todo->id }})">
    <span :class="done && 'line-through'">{{ $todo->title }}</span>
</div>
```

## 주의사항 / 안티패턴

- v3 웹뷰용으로 짠 `wire:navigate` 기반 페이지 전환 로직을 v4 EDGE로 그대로 옮기면 동작하지 않는다 — EDGE는 [[NativePHP Mobile Routing and Navigation]]의 `$this->navigate()`/`Route::native()` 기반 네이티브 내비게이션을 쓴다.
- 모바일에서 public 프로퍼티에 큰 컬렉션을 담는 실수([[Livewire Common Pitfalls]] 4번)는 데스크톱보다 체감 성능 저하가 크다 — `#[Computed]`를 더 적극적으로 쓴다.
- NativePHP 자체의 라이선스·PHP 버전 고정·성숙도(breaking change 가능성) 관련 주의사항은 [[NativePHP Mobile Practical Notes]]에 정리되어 있다 — 프로덕션 도입 전 반드시 확인한다.

## 참고

- [[Livewire Overview]] — NativePHP에서 Livewire의 네트워크 왕복 약점이 사라지는 이유
- [[Livewire Alpine Integration]] — 모바일 터치 반응에 Alpine을 우선 적용하는 판단 기준
- [[Livewire Advanced v4 Features]] — Islands/Lazy로 저사양 기기 초기 렌더링 부담 완화
- [[Livewire Common Pitfalls]] — 모바일에서 특히 체감되는 함정들
- [[NativePHP Mobile Overview]] — NativePHP 자체의 개념과 Laravel 대응 관계
- [[NativePHP SuperNative Architecture]] — v4 아키텍처(공유 메모리, EDGE 컴포넌트) 상세
- [[NativePHP Mobile Practical Notes]] — 라이선스/PHP 버전/성숙도 주의사항
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`, `2026-08-12_NativePHP for Mobile 실무 가이드.md`

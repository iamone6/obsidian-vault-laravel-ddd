---
title: NativePHP Mobile Overview
category: frontend
tags: [frontend, nativephp, mobile, laravel]
related: [[NativePHP SuperNative Architecture]], [[NativePHP Mobile Environment Setup]], [[Livewire and NativePHP]]
---

# NativePHP Mobile Overview

NativePHP for Mobile은 **Laravel 앱을 iOS/Android 네이티브 앱으로 만들어 주는 Composer 패키지**다. 기준 버전 v4.

## 핵심 개념

세 가지 핵심 아이디어.

1. **웹서버가 없다.** 앱 안에 PHP 런타임이 통째로 내장(embed)된다. 서버에 요청을 보내는 게 아니라, 기기 안에서 PHP가 직접 돈다. 완전한 오프라인 동작이 기본값이다.
2. **v4부터는 웹뷰가 기본이 아니다.** v3까지는 "네이티브 껍데기 + 웹뷰에 Blade 렌더링" 구조였지만, v4의 **SuperNative** 아키텍처는 Blade 문법으로 **진짜 네이티브 UI 위젯**(SwiftUI / Jetpack Compose)을 그린다. `<native:button>`은 스타일링된 `<div>`가 아니라 그 기기의 실제 네이티브 버튼이다. 자세한 건 [[NativePHP SuperNative Architecture]].
3. **웹뷰도 여전히 쓸 수 있다.** 기본값이 아니라 "컴포넌트 중 하나"로 강등됐을 뿐이다. 기존 웹뷰 방식 앱을 그대로 유지하면서 화면 단위로 하나씩 네이티브로 옮겨갈 수 있다.

### Laravel 개발자 입장에서의 대응 관계

| Laravel 웹 | NativePHP Mobile v4 |
|---|---|
| `routes/web.php` + `Route::get()` | `routes/mobile.php` + `Route::native()` |
| Livewire 컴포넌트 | `NativeComponent` (거의 동일한 감각) |
| Blade 뷰 + HTML 태그 | Blade 뷰 + `<native:*>` EDGE 컴포넌트 |
| `redirect()->route()` | `$this->navigate('/path')` |
| 브라우저 뒤로가기 | 네이티브 내비게이션 스택 (`$this->back()`) |
| `php artisan serve` | `php artisan native:run` |

## Laravel 구현

`NativeComponent`는 Livewire 컴포넌트와 거의 동일한 감각으로 설계됐다 — 프로퍼티가 상태, 메서드가 액션, `render()`가 Blade 뷰를 반환한다. [[Livewire and NativePHP]]에서 다뤘듯 Livewire를 먼저 익혀두면 이 전환에 학습 비용이 거의 들지 않는다.

## 주의사항 / 안티패턴

- v3 웹뷰 방식과 v4 SuperNative(EDGE) 방식은 아키텍처가 근본적으로 다르다 — v3용으로 짠 코드를 그대로 v4에 옮길 수 없다. 새 프로젝트라면 SuperNative를 기본으로 검토한다.
- 이 위키의 다른 페이지(예: [[Livewire and NativePHP]])에서는 v4를 "beta"로 언급하기도 하는데, 프로덕션 도입 전에는 [[NativePHP Mobile Practical Notes]]의 성숙도/breaking change 주의사항을 반드시 확인한다.

## 참고

- [[NativePHP SuperNative Architecture]] — v4 아키텍처 상세
- [[NativePHP Mobile Environment Setup]] — 개발 환경 준비
- [[NativePHP Mobile Routing and Navigation]] — `Route::native()`, `$this->navigate()`
- [[Livewire and NativePHP]] — Livewire 학습이 NativePHP로 이어지는 이유
- 소스: `2026-08-12_NativePHP for Mobile 실무 가이드.md`

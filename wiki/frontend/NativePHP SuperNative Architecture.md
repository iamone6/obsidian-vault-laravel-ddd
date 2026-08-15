---
title: NativePHP SuperNative Architecture
category: frontend
tags: [frontend, nativephp, mobile, architecture]
related: [[NativePHP Mobile Overview]], [[Livewire and NativePHP]], [[NativePHP EDGE Components]]
---

# NativePHP SuperNative Architecture

NativePHP for Mobile v4의 핵심 아키텍처. 세 가지 아이디어가 맞물려 돌아간다.

## 핵심 개념

### PHP와 네이티브 레이어의 공유 메모리

네이티브 레이어와 PHP 애플리케이션이 **메모리를 직접 공유**한다. 네트워크 왕복도, 직렬화 오버헤드도, 웹뷰 브리지 대기도 없다. 상태 변경이 PHP와 네이티브 UI 사이를 거의 즉시 오간다.

Laravel 관점에서 비유하면, HTTP 요청/응답 사이클을 거치는 Livewire와 달리 **같은 프로세스 메모리 안에서 컴포넌트 상태와 뷰가 붙어 있는** 구조다. Livewire의 최대 약점인 네트워크 왕복이 여기서는 아예 존재하지 않는다 — [[Livewire and NativePHP]]에서 다룬 "NativePHP에서는 왕복이 localhost 호출이라 지연이 거의 0"이라는 설명보다도 한 단계 더 나아간 것이, v4 SuperNative는 애초에 네트워크 계층 자체가 없다는 점이다.

### Livewire 유사 컴포넌트

각 화면이 `NativeComponent` 하나로 구동된다. 프로퍼티가 상태이고, 메서드가 액션이며, `render()`가 Blade 뷰를 반환한다. Livewire를 써봤다면 학습 곡선이 거의 없다.

### EDGE 컴포넌트

`<native:button>`, `<native:list>` 같은 태그가 플랫폼 고유 UI 프레임워크의 실제 위젯으로 매핑된다. 스크롤 물리, 텍스트 렌더링, 전환 애니메이션, 컨텍스트 메뉴, 다크 모드, 다이내믹 타입, 스크린 리더 — 브라우저에서 흉내 내야 했던 것들이 플랫폼 네이티브에서는 기본으로 제공된다. 전체 목록은 [[NativePHP EDGE Components]] 참고.

## Laravel 구현

### 웹뷰 방식 그대로 유지하기 (마이그레이션 전략)

v3 이하 방식 앱을 그대로 살리려면:

```php
// routes/mobile.php
Route::native('/home', WebViewScreen::class);
```

```blade
{{-- resources/views/webviewscreen.blade.php --}}
<webview php url="/" fullscreen />
```

```php
// routes/web.php
Route::view('/', 'welcome');
```

```dotenv
NATIVEPHP_START_URL=/home
```

이렇게 두면 기존 웹뷰 기반 앱이 그대로 동작하고, 준비될 때 화면 단위로 하나씩 SuperNative로 전환할 수 있다. 전환하지 않아도 무방하다.

## 주의사항 / 안티패턴

- SuperNative의 공유 메모리 모델은 웹뷰 방식과 근본적으로 다른 실행 모델이다 — 웹뷰 시절의 JS 브리지 호출 패턴을 그대로 이식하려 하지 말고, `NativeComponent`의 프로퍼티/메서드 모델로 다시 설계하는 편이 낫다.
- 마이그레이션을 전부 한 번에 하려 하지 말 것 — `<webview>` 컴포넌트로 기존 화면을 유지하면서 화면 단위로 점진 전환하는 것이 공식 권장 경로다.

## 참고

- [[NativePHP Mobile Overview]] — 전체 개념과 Laravel 대응 관계
- [[NativePHP EDGE Components]] — `<native:*>` 컴포넌트 전체 목록
- [[Livewire and NativePHP]] — Livewire와 NativeComponent의 유사성
- 소스: `2026-08-12_NativePHP for Mobile 실무 가이드.md`

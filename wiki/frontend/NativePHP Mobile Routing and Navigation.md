---
title: NativePHP Mobile Routing and Navigation
category: frontend
tags: [frontend, nativephp, mobile, routing]
related: [[NativePHP SuperNative Architecture]], [[NativePHP EDGE Components]], [[Livewire URL and Navigation]]
---

# NativePHP Mobile Routing and Navigation

네이티브 화면은 iOS `UINavigationController` / Android Activity 스택처럼 **스택 구조**를 이룬다. `Route::native()`로 등록하고 `$this->navigate()` 계열 API로 이동한다.

## 핵심 개념

### 라우트 등록

```php
// routes/mobile.php
use App\NativeComponents\Home;
use App\NativeComponents\ItemDetail;

Route::native('/', Home::class);
Route::native('/item/{id}', ItemDetail::class);
```

Livewire의 `Route::livewire()` 자리에 `Route::native()` 매크로를 쓴다고 보면 된다. 라우트 파라미터는 Laravel 웹 라우트와 동일하게 동작한다.

### 화면 이동 API

```php
$this->navigate('/item/42');                              // 새 화면 push (기본: 오른쪽에서 슬라이드 인)
$this->navigate('/item/42', ['source' => 'home-feed']);   // 데이터 전달
$this->back();                                              // 뒤로 가기 (스택에서 pop)
$this->replace('/login');                                   // 현재 화면 교체 (기본: 페이드)
$this->exitToWeb('/dashboard');                              // 네이티브 스택을 걷어내고 웹뷰로 진입
```

### 파라미터와 데이터 읽기

```php
class ItemDetail extends NativeComponent
{
    public function mount(): void
    {
        $id     = $this->param('id');                  // 라우트 URI에서
        $source = $this->data('source', 'unknown');    // navigate() 두 번째 인자에서
    }
}
```

두 접근자 모두 키가 없을 때 반환할 기본값을 두 번째 인자로 받는다.

### 이름 있는 라우트

```php
$this->navigate($this->route('listing.show', ['id' => 5]));
```

Laravel의 URL 제너레이터에 `absolute: false`로 위임한다.

## Laravel 구현

### 전환 애니메이션

```php
use Native\Mobile\Edge\Transition;

$this->navigate('/item/42')->transition(Transition::SlideFromBottom);
```

| 값 | 설명 |
|---|---|
| `SlideFromRight` | `navigate()` 기본값 |
| `SlideFromLeft` | `back()` 기본값 |
| `SlideFromBottom` | 모달 스타일 표현 |
| `Fade` | `replace()` 기본값 |
| `FadeFromBottom` | 은은한 수직 페이드 |
| `ScaleFromCenter` | 줌인 효과 |
| `ParallaxPush` | 들어오는 화면은 우측에서, 나가는 화면은 좌측으로 밀리는 패럴랙스 |
| `None` | 애니메이션 없음 |

### Blade에서 직접 내비게이션

컴포넌트 메서드를 만들 필요 없이 `@navigate` 디렉티브로 처리할 수 있다. 어떤 노드에든 붙일 수 있어 모든 요소가 내비게이션 타깃이 된다.

```blade
<native:pressable @navigate="/item/42" class="w-full p-4">
    <native:text>항목 보기</native:text>
</native:pressable>

{{-- 데이터 전달 --}}
<native:pressable @navigate="'/item/42', ['source' => 'home-feed']">...</native:pressable>

{{-- 전환 지정 --}}
<native:pressable @navigate.slideFromBottom="/item/42">모달로 열기</native:pressable>

{{-- 액션 지정 --}}
<native:pressable @navigate.back>뒤로</native:pressable>
<native:pressable @navigate.replace="/login">로그아웃</native:pressable>
<native:pressable @navigate.exitToWeb="/dashboard">대시보드 열기</native:pressable>
```

전환 modifier와 액션 modifier는 조합할 수 있다 — `@navigate.replace.fade="/login"`. 전환 modifier 목록: `fade`, `slideFromRight`, `slideFromLeft`, `slideFromBottom`, `fadeFromBottom`, `scaleFromCenter`, `parallaxPush`, `none`.

### Android 하드웨어 뒤로가기 제어

기본적으로 기기 뒤로가기 버튼은 스택을 pop한다. `onBackPressed()`를 오버라이드해 커스텀 동작을 넣을 수 있다.

```php
class CheckoutForm extends NativeComponent
{
    public bool $dirty = false;

    public function onBackPressed(): void
    {
        if ($this->dirty) {
            $this->showDiscardConfirmation();
            return;
        }

        $this->back();
    }
}
```

오버라이드해도 최종적으로 pop해야 한다면 `$this->back()`을 직접 호출해야 한다.

> `<native:bottom-nav-item>` 탭은 자동으로 `replace` 시맨틱을 쓴다. 그래서 뒤로가기가 탭 히스토리를 하나씩 되짚지 않고 탭 섹션 전체를 한 번에 빠져나온다.

## 주의사항 / 안티패턴

- `onBackPressed()`를 오버라이드하고 조건부로 처리할 때 `$this->back()` 호출을 빠뜨리면, 확인창을 닫아도 화면이 스택에서 pop되지 않아 사용자가 갇힌 것처럼 느낄 수 있다.
- 웹의 URL 쿼리스트링 감각으로 `#[Url]`([[Livewire URL and Navigation]])을 그대로 쓰려 하지 말 것 — 모바일 네이티브 화면은 URL 개념이 약하고, `param()`/`data()`로 명시적으로 값을 주고받는 방식이 기본이다.

## 참고

- [[NativePHP SuperNative Architecture]] — `NativeComponent`의 전체 구조
- [[NativePHP EDGE Components]] — `native:pressable` 등 내비게이션에 쓰이는 컴포넌트
- [[Livewire URL and Navigation]] — 웹 Livewire의 `wire:navigate`와의 대응/차이
- 소스: `2026-08-12_NativePHP for Mobile 실무 가이드.md`

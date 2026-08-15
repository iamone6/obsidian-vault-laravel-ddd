---
title: Livewire Common Pitfalls
category: frontend
tags: [frontend, livewire, laravel, anti-pattern]
related: [[Livewire Rendering and wire-key]], [[Livewire Render Cycle]], [[Livewire Actions]]
---

# Livewire Common Pitfalls

실무에서 반복적으로 겪는 Livewire 함정 모음. 대부분 [[Livewire Render Cycle]]의 4가지 결론과 [[Livewire Rendering and wire-key]]의 morphing 규칙으로 설명된다.

## 핵심 개념

### 1. 루트 엘리먼트가 없거나 둘 이상

```blade
{{-- ❌ 아무것도 안 보이거나 이상하게 동작 --}}
<h1>제목</h1>
<p>내용</p>

{{-- ✅ --}}
<div><h1>제목</h1><p>내용</p></div>
```

### 2. `@if`로 루트를 감싸기

```blade
{{-- ❌ 조건이 거짓이면 루트가 사라져 컴포넌트가 죽음 --}}
@if ($show)
    <div>...</div>
@endif

{{-- ✅ --}}
<div>
    @if ($show) ... @endif
</div>
```

### 3. 반복문에 `wire:key` 누락

증상: 삭제했는데 다른 항목이 사라진 것처럼 보임, input 값이 뒤섞임. → [[Livewire Rendering and wire-key]]

### 4. public 프로퍼티에 큰 컬렉션 담기

```php
public Collection $posts;                          // ❌ 매 요청마다 왕복
#[Computed] public function posts() { return Post::all(); }   // ✅
```

### 5. `private`/`protected` 프로퍼티가 사라짐

```php
// ❌ mount()에서 세팅해도 다음 요청에서 null
protected $service;
public function mount() { $this->service = app(Service::class); }

// ✅ boot()에서 매 요청 세팅
public function boot() { $this->service = app(Service::class); }
```

### 6. 모델의 관계가 사라짐

```php
public function mount(Post $post) { $this->post = $post->load('comments'); }  // ❌
#[Computed] public function comments() { return $this->post->comments()->get(); }  // ✅
```

### 7. `wire:model` 기본값이 지연이라는 걸 모름

```blade
{{-- 타이핑해도 서버로 안 감. 실시간을 원하면 --}}
<input wire:model.live.debounce.300ms="search">
```

### 8. 액션 파라미터를 신뢰

```php
public function delete(int $id) { Post::find($id)->delete(); }  // ❌

public function delete(Post $post)                                // ✅
{
    $this->authorize('delete', $post);
    $post->delete();
}
```

## Laravel 구현

### 9. 배열에서 `unset()` 후 `array_values()` 누락

```php
unset($this->items[$i]);
$this->items = array_values($this->items);   // 필수
```

### 10. 서드파티 JS가 깨짐

```blade
<div wire:ignore>
    <select id="select2">...</select>
</div>
```

### 11. `wire:navigate` 사용 시 스크립트 재실행 안 됨

```js
document.addEventListener('DOMContentLoaded', init);      // ❌
document.addEventListener('livewire:navigated', init);    // ✅
```

### 12. 예약어 메서드명 사용

`reset`, `mount`, `render`, `validate`, `dispatch`, `fill`, `all`, `boot` 등은 피한다.

### 13. 페이지 하나에 페이지네이션이 둘

```php
Post::paginate(10, pageName: 'postsPage');
Comment::paginate(10, pageName: 'commentsPage');
```

### 14. v4 컴포넌트 태그를 안 닫음

```blade
<livewire:counter />   {{-- ✅ --}}
<livewire:counter>     {{-- ❌ v4에서는 뒤 내용이 slot으로 해석됨 --}}
```

### 15. `/livewire/` 경로 하드코딩

v4부터 엔드포인트가 `/livewire-{hash}/`로 바뀌었다. 방화벽 규칙, CDN 설정, 미들웨어에 `/livewire/`를 하드코딩했다면 수정이 필요하다.

## 주의사항 / 안티패턴

이 페이지 자체가 안티패턴 모음이다. 특히 3번(`wire:key`)과 8번(액션 파라미터 신뢰)은 각각 UI 버그와 보안 취약점으로 직결되므로 코드 리뷰 체크리스트에 포함하는 것을 권장한다.

## 참고

- [[Livewire Render Cycle]] — 5·6번 함정의 근본 원인(인스턴스 비영속성)
- [[Livewire Rendering and wire-key]] — 3번 함정의 상세
- [[Livewire Actions]] — 8번 함정과 `authorize()` 패턴
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

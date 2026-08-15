---
title: Livewire Computed Properties
category: frontend
tags: [frontend, livewire, laravel, performance]
related: [[Livewire Render Cycle]], [[Livewire Rendering and wire-key]], [[Livewire Properties]]
---

# Livewire Computed Properties

`render()`가 매 요청마다 통째로 재실행되는 [[Livewire Render Cycle]]의 결론 4를 해결하는 도구. DB 조회를 요청 내에서 메모이제이션한다.

## 핵심 개념

### 문제 상황

```php
public function render()
{
    return $this->view([
        // ❌ 버튼을 하나 눌러도 이 쿼리가 매번 실행됨
        'posts' => Post::with('author')->where(...)->get(),
    ]);
}
```

### 해결

```php
use Livewire\Attributes\Computed;

new class extends Component {
    public string $search = '';

    #[Computed]
    public function posts()
    {
        return Post::with('author')->where('title', 'like', "%{$this->search}%")->latest()->get();
    }
};
```

```blade
{{-- $this-> 를 반드시 붙여야 함 --}}
@foreach ($this->posts as $post)
    <article wire:key="{{ $post->id }}">{{ $post->title }}</article>
@endforeach

<p>총 {{ $this->posts->count() }}건</p>
{{-- 여러 번 써도 쿼리는 1회만 실행 (요청 내 메모이제이션) --}}
```

**핵심**: 한 요청 안에서 여러 번 접근해도 **첫 호출 결과를 재사용**한다.

## Laravel 구현

### 캐시 옵션

```php
// 요청 간에도 유지 (컴포넌트가 살아있는 동안)
#[Computed(persist: true)]
public function user() { return User::find($this->userId); }

// 캐시 지속 시간 지정 (초)
#[Computed(persist: true, seconds: 3600)]
public function stats() { /* ... */ }

// Laravel 캐시 드라이버 사용 (모든 사용자 공유)
#[Computed(cache: true)]
public function popularTags()
{
    return Tag::withCount('posts')->orderByDesc('posts_count')->take(10)->get();
}

// 캐시 키 지정
#[Computed(cache: true, key: 'popular-tags')]
public function popularTags() { /* ... */ }
```

### 캐시 무효화

```php
public function addPost()
{
    Post::create([...]);
    unset($this->posts);          // 캐시 삭제 → 다음 접근 시 재계산
}
```

### PHP 쪽에서 사용

```php
public function publishAll()
{
    // 메서드 호출이 아니라 프로퍼티 접근처럼
    foreach ($this->posts as $post) { $post->publish(); }
}
```

### 언제 쓰나 — 판단 기준

| 상황 | 방법 |
|---|---|
| DB 조회 결과 | `#[Computed]` |
| 단순 계산 (합계 등) | `#[Computed]` 또는 Blade 안에서 직접 |
| 사용자 입력값 | public 프로퍼티 |
| 요청 사이에 유지되어야 하는 상태 | public 프로퍼티 |
| 매번 최신값이어야 하는 것 | `render()` 안에서 조회 |

> **왜 컬렉션을 public 프로퍼티에 담으면 안 되나**
> ```php
> public Collection $posts;   // ❌ 매 요청마다 전체가 JSON으로 왕복
> ```
> 게시글 100건이면 100건의 데이터가 매 클릭마다 브라우저 ↔ 서버를 왕복한다. `#[Computed]`는 서버에서만 계산되고 HTML만 나간다.

## 주의사항 / 안티패턴

- `{{ $this->posts }}`처럼 반드시 `$this->`를 붙여야 한다 — public 프로퍼티처럼 `{{ $posts }}`로 접근할 수 없다.
- 다른 컴포넌트의 이벤트로 데이터가 바뀌었는데 `unset($this->posts)`로 캐시를 무효화하지 않으면 화면이 갱신되지 않는다 ([[Livewire Events]]의 "실전 패턴 2" 참고).
- `#[Computed(cache: true)]`는 모든 사용자가 공유하는 캐시이므로 사용자별로 달라야 하는 데이터에 쓰면 다른 사용자의 데이터가 노출될 수 있다 — `key`에 사용자 식별자를 포함시켜야 한다.

## 참고

- [[Livewire Render Cycle]] — `render()`가 매번 재실행되는 이유
- [[Livewire Rendering and wire-key]] — computed 결과를 렌더링할 때의 `wire:key` 규칙
- [[Livewire Properties]] — public 프로퍼티와의 선택 기준
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

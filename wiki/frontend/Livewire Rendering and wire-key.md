---
title: Livewire Rendering and wire:key
category: frontend
tags: [frontend, livewire, laravel, morphing]
related: [[Livewire Render Cycle]], [[Livewire Computed Properties]], [[Blade Includes and Loops]]
---

# Livewire Rendering and wire:key

`render()`가 뷰에 데이터를 넘기는 방법, morphing 알고리즘, 그리고 반복문에서 **반드시** 알아야 하는 `wire:key` 규칙.

## 핵심 개념

### `render()` 메서드

**SFC**에서는 `render()`가 없어도 된다. 파일 아래쪽 HTML이 자동으로 뷰가 된다. 뷰에 데이터를 넘기고 싶을 때만 정의한다.

```php
public function render()
{
    return $this->view(['author' => Auth::user(), 'currentTime' => now()]);
}
```

**Class-based**에서는 필수다: `return view('livewire.post-list', ['posts' => Post::latest()->get()]);`

### 뷰에 데이터를 주는 3가지 방법 비교

```php
new class extends Component {
    // 1. public 프로퍼티 — 상태. 매 요청 왕복.
    public string $search = '';

    // 2. computed — 서버 계산. 왕복 없음. $this-> 로 접근.
    #[Computed]
    public function posts() { return Post::all(); }

    // 3. render() 전달 — 매 요청 재계산. 왕복 없음. 바로 접근.
    public function render() { return $this->view(['now' => now()]); }
};
```

```blade
{{ $search }}         {{-- 1 --}}
{{ $this->posts }}    {{-- 2 --}}
{{ $now }}            {{-- 3 --}}
```

### Morphing과 `wire:key` — 반드시 이해할 것

Livewire는 새 HTML과 기존 DOM을 **위에서부터 순서대로 비교**한다. 리스트에서 항목이 추가/삭제/정렬되면 이 비교가 어긋나 엉뚱한 결과가 나온다.

```blade
{{-- ❌ 삭제하면 엉뚱한 항목이 사라진 것처럼 보임 --}}
@foreach ($posts as $post)
    <div>
        <input wire:model="titles.{{ $post->id }}">
        <button wire:click="delete({{ $post->id }})">삭제</button>
    </div>
@endforeach

{{-- ✅ --}}
@foreach ($posts as $post)
    <div wire:key="post-{{ $post->id }}">
        <input wire:model="titles.{{ $post->id }}">
        <button wire:click="delete({{ $post->id }})">삭제</button>
    </div>
@endforeach
```

**규칙**: 반복문 안의 최상위 엘리먼트에는 **항상** `wire:key`를 붙인다. 값은 그 항목의 고유 ID여야 한다 (`$loop->index`는 순서가 바뀌면 무의미하므로 부적합 — [[Blade Includes and Loops]]의 `$loop` 참고).

중첩 컴포넌트도 마찬가지다.

```blade
@foreach ($posts as $post)
    <livewire:post-card :post="$post" wire:key="card-{{ $post->id }}" />
@endforeach
```

> v4에서 `smart_wire_keys`가 기본 `true`가 되어 깊게 중첩된 컴포넌트의 키 문제는 줄었지만, **반복문에서는 여전히 직접 붙여야 한다.**

## Laravel 구현

### `wire:ignore` — DOM을 건드리지 않게

Select2, Flatpickr, TinyMCE 같은 JS 라이브러리가 조작한 DOM은 Livewire가 갈아엎으면 깨진다.

```blade
<div wire:ignore>
    <select id="my-select2">...</select>
</div>

<div wire:ignore.self>   {{-- 자기 자신만 제외, 자식은 갱신 허용 --}}
    <span>{{ $count }}</span>
</div>
```

### `wire:replace` (v4)

morphing 대신 **통째로 교체**하게 한다. 애니메이션 재시작 등에 유용하다.

```blade
<div wire:replace>
    <div class="animate-fade-in">{{ $message }}</div>
</div>
```

### `wire:poll` — 주기적 갱신

```blade
<div wire:poll>...</div>                          {{-- 2.5초마다 (기본값) --}}
<div wire:poll.5s>...</div>
<div wire:poll.10s="refreshStats">...</div>       {{-- 특정 메서드 호출 --}}
<div wire:poll.visible>...</div>                   {{-- 탭이 보일 때만 --}}
<div wire:poll.keep-alive>...</div>                {{-- 백그라운드에서도 계속 --}}
```

> v4에서 폴링은 **논블로킹**이다. 다른 요청을 막지 않고, 막히지도 않는다.

## 주의사항 / 안티패턴

- 반복문에서 `wire:key`를 빠뜨리면 항목을 삭제했는데 다른 항목이 사라진 것처럼 보이거나 input 값이 뒤섞이는 증상이 나타난다 — Livewire에서 가장 흔한 버그다.
- `wire:key` 값으로 배열 인덱스(`$loop->index`)를 쓰지 말 것 — 정렬/삭제로 순서가 바뀌면 키가 항목을 따라가지 못한다. 반드시 DB의 고유 ID를 쓴다.
- 서드파티 JS가 DOM을 직접 조작하는 영역에 `wire:ignore`를 빠뜨리면 다음 재렌더링에서 그 상태가 초기화된다.

## 참고

- [[Livewire Render Cycle]] — morphing이 일어나는 전체 사이클
- [[Livewire Computed Properties]] — `render()` 대신 계산을 옮기는 방법
- [[Blade Includes and Loops]] — `$loop` 객체와 반복문 일반
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

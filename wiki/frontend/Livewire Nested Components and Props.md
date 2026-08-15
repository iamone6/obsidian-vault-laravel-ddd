---
title: Livewire Nested Components and Props
category: frontend
tags: [frontend, livewire, laravel, component]
related: [[Livewire Events]], [[Blade Component Basics]], [[Livewire and NativePHP]]
---

# Livewire Nested Components and Props

자식 컴포넌트에 값을 넘기는 방법과, "props는 기본적으로 반응형이 아니다"라는 가장 헷갈리는 함정.

## 핵심 개념

```blade
<livewire:post-card :post="$post" wire:key="{{ $post->id }}" />
<livewire:user-badge name="홍길동" :admin="true" />
<livewire:post-card :$post :$showActions />   {{-- 여러 값 한 번에 --}}
```

받는 쪽:

```php
new class extends Component {
    public Post $post;
    public bool $showActions = false;
    // mount() 생략 가능 — 이름이 일치하면 자동 주입
};
```

### props는 기본적으로 반응형이 아니다

**흔히 헷갈리는 부분이다.**

```blade
<livewire:child :count="$parentCount" />
```

부모의 `$parentCount`가 바뀌어도 자식의 `$count`는 **자동으로 바뀌지 않는다.** 최초 마운트 시점의 값만 전달된다. React/Vue의 props와 다른 지점이다 — Livewire 자식 컴포넌트는 자기만의 독립된 생명주기와 snapshot을 가지기 때문이다.

### `#[Reactive]` — 반응형 props (v3+)

```php
use Livewire\Attributes\Reactive;

new class extends Component {
    #[Reactive]
    public int $count;    // 부모가 바뀌면 자식도 갱신됨
};
```

> ⚠️ 반응형 props는 부모가 재렌더링될 때마다 자식도 재렌더링된다. 남용하면 성능이 떨어진다.

### `#[Modelable]` — 자식과 양방향 바인딩

커스텀 input 컴포넌트를 만들 때 쓴다.

```php
{{-- resources/views/components/⚡text-input.blade.php --}}
use Livewire\Attributes\Modelable;

new class extends Component {
    #[Modelable]
    public string $value = '';
    public string $label = '';
};
```
```blade
<div>
    <label>{{ $label }}</label>
    <input type="text" wire:model.live="value">
</div>
```

부모에서: `<livewire:text-input wire:model="title" label="제목" />` — 자식의 `$value`가 부모의 `$title`과 연결된다.

## Laravel 구현

### `$parent` — 부모 메서드 호출

```blade
<button wire:click="$parent.removeItem({{ $item->id }})">삭제</button>
```

```php
// 자식 PHP에서
public function someAction() { $this->dispatch('item-removed')->to('parent-component'); }
```

### Slots (v4 신기능)

```blade
{{-- 사용하는 쪽 --}}
<livewire:card title="공지사항">
    <p>내용이 여기 들어갑니다.</p>
    <x-slot:footer><button>확인</button></x-slot:footer>
</livewire:card>
```

```blade
{{-- card 컴포넌트 --}}
<div class="border rounded p-4" {{ $attributes }}>
    <h3>{{ $title }}</h3>
    <div>{{ $slot }}</div>
    @isset($footer)<div class="mt-4 border-t pt-2">{{ $footer }}</div>@endisset
</div>
```

`{{ $attributes }}`로 속성 전달(attribute forwarding)도 된다: `<livewire:card class="shadow-lg" data-testid="card" />`. Slot/`$attributes` 문법 자체는 [[Blade Component Basics]], [[Blade Component Attributes]]와 동일하다.

### 중첩 구조 설계 가이드

| 상황 | 권장 |
|---|---|
| 단순 반복 렌더링 (표시만) | Blade 컴포넌트 (`<x-post-card />`) |
| 각 항목이 자체 상태를 가짐 (인라인 편집 등) | Livewire 컴포넌트 |
| 항목이 100개 이상 | Blade 컴포넌트 — Livewire 컴포넌트 100개는 무겁다 |
| 무거운 위젯 일부만 갱신 | Islands ([[Livewire Advanced v4 Features]]) |

> **성능 주의**: Livewire 컴포넌트 하나하나가 자체 snapshot을 가진다. 리스트의 각 행을 Livewire 컴포넌트로 만들면 페이지 하나에 snapshot이 수십 개 생긴다. 웬만하면 Blade 컴포넌트를 쓰고, 정말 독립적 상태가 필요할 때만 Livewire로 만든다.

## 주의사항 / 안티패턴

- 부모 상태가 바뀌면 자식도 당연히 바뀔 거라 기대하지 말 것 — 반응형이 필요하면 `#[Reactive]`를 명시적으로 붙여야 한다.
- 리스트의 각 항목을 전부 Livewire 컴포넌트로 만들면(특히 100개 이상) snapshot 수가 폭증해 페이지가 무거워진다 — 표시만 하는 항목은 Blade 컴포넌트로 충분하다.
- 자식 컴포넌트에 `wire:key`를 빠뜨리면 v4의 `smart_wire_keys`로도 완전히 커버되지 않는 경우가 있다 — 반복문 안에서는 여전히 직접 붙인다 ([[Livewire Rendering and wire-key]]).

## 참고

- [[Livewire Events]] — 컴포넌트 간 통신의 다른 방법(이벤트 vs `$parent`)
- [[Blade Component Basics]] — Slot/`$attributes` 문법의 원형
- [[Livewire Advanced v4 Features]] — Islands로 무거운 위젯 일부만 갱신
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

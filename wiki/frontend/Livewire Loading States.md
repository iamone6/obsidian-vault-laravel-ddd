---
title: Livewire Loading States
category: frontend
tags: [frontend, livewire, laravel, ux]
related: [[Livewire Actions]], [[Livewire Forms and Validation]], [[Tailwind Component Strategy]]
---

# Livewire Loading States

모든 상호작용이 네트워크를 타는 Livewire에서, "지금 처리 중"임을 보여주는 것은 특히 중요하다.

## 핵심 개념

```blade
<div wire:loading>불러오는 중...</div>       {{-- 요청 중일 때만 표시 --}}
<div wire:loading.remove>내용</div>          {{-- 요청 중일 때 숨김 --}}
```

### 특정 액션만 대상으로

```blade
<span wire:loading wire:target="save">저장 중...</span>
<span wire:loading wire:target="save, delete">처리 중...</span>            {{-- 여러 개 --}}
<span wire:loading wire:target="delete(3)">3번 삭제 중...</span>            {{-- 파라미터까지 지정 --}}
<span wire:loading wire:target="search">검색 중...</span>                   {{-- 특정 프로퍼티 갱신 중 --}}
<span wire:loading wire:target.except="poll">처리 중...</span>              {{-- 특정 대상 제외 --}}
```

### 속성/클래스 조작

```blade
<button wire:click="save" wire:loading.attr="disabled">저장</button>
<button wire:click="save" wire:loading.class="opacity-50 cursor-wait">저장</button>
<button wire:click="save" wire:loading.class.remove="bg-blue-600">저장</button>
```

### `data-loading` 속성 (v4)

v4부터 요청을 유발하는 모든 엘리먼트에 자동으로 `data-loading` 속성이 붙는다. Tailwind와 조합하면 간결해진다 ([[Tailwind Component Strategy]] 참고).

```blade
<button wire:click="save"
        class="bg-blue-600 text-white px-4 py-2 rounded
               data-loading:opacity-50 data-loading:pointer-events-none">
    저장
</button>
```

### 지연 표시 (깜빡임 방지)

빠른 요청에서 로딩 인디케이터가 번쩍이는 걸 막는다.

```blade
<div wire:loading.delay>로딩...</div>          {{-- 200ms 후 --}}
<div wire:loading.delay.shortest>...</div>     {{-- 50ms --}}
<div wire:loading.delay.short>...</div>        {{-- 150ms --}}
<div wire:loading.delay.long>...</div>         {{-- 300ms --}}
<div wire:loading.delay.longest>...</div>      {{-- 1000ms --}}
```

## Laravel 구현

### 실전 예시 — 스켈레톤 + 버튼

```blade
<form wire:submit="search" class="flex gap-2">
    <input type="text" wire:model="query" class="border rounded p-2 flex-1">
    <button type="submit" wire:loading.attr="disabled" wire:target="search"
            class="bg-blue-600 text-white px-4 py-2 rounded disabled:opacity-50">
        <span wire:loading.remove wire:target="search">검색</span>
        <span wire:loading wire:target="search">검색 중</span>
    </button>
</form>

{{-- 스켈레톤 --}}
<div wire:loading.delay wire:target="search" class="mt-4 space-y-2">
    @for ($i = 0; $i < 3; $i++)
        <div class="h-16 bg-gray-200 rounded animate-pulse"></div>
    @endfor
</div>

{{-- 결과 --}}
<div wire:loading.remove wire:target="search" class="mt-4">
    @foreach ($this->results as $result)
        <div wire:key="{{ $result->id }}">{{ $result->title }}</div>
    @endforeach
</div>
```

### 기타 상태 디렉티브

```blade
<div wire:offline class="bg-red-100 p-2">인터넷 연결이 끊겼습니다</div>

{{-- 값이 변경되었지만 아직 저장 안 됨 --}}
<span wire:dirty wire:target="title">변경됨</span>
<input wire:model="title" wire:dirty.class="border-yellow-500">

<div wire:cloak>...</div>   {{-- Alpine 초기화 전 깜빡임 방지 --}}
```

## 주의사항 / 안티패턴

- `wire:target`을 지정하지 않으면 페이지의 **아무 요청**에나 로딩 표시가 반응한다 — 저장 버튼에 검색 요청의 로딩이 뜨는 식으로 헷갈릴 수 있다.
- 저장 버튼에 `wire:loading.attr="disabled"` + `wire:target`을 빠뜨리면 느린 네트워크에서 중복 클릭 → 중복 제출로 이어질 수 있다 ([[Livewire Forms and Validation]] 참고).
- `wire:loading.delay` 없이 즉시 표시하면 빠른 요청에서도 인디케이터가 한 프레임 번쩍인다.

## 참고

- [[Livewire Actions]] — `wire:target`이 가리키는 액션
- [[Livewire Forms and Validation]] — 저장 중 중복 제출 방지
- [[Tailwind Component Strategy]] — `data-loading:` 유틸리티 조합
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

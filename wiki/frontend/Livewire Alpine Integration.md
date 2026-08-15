---
title: Livewire Alpine Integration
category: frontend
tags: [frontend, livewire, alpinejs, laravel]
related: [[Livewire Overview]], [[Livewire and NativePHP]], [[Livewire Nested Components and Props]]
---

# Livewire Alpine Integration

Livewire에는 Alpine.js가 기본 포함되어 있다. **서버 왕복이 필요 없는 UI 상호작용은 Alpine으로 처리**하는 것이 원칙이다.

## 핵심 개념

### 기본 조합

```blade
<div x-data="{ open: false }">
    <button @click="open = !open">토글</button>
    <div x-show="open" x-transition>서버 요청 없이 즉시 열고 닫힘</div>
</div>
```

### `$wire` — Alpine에서 Livewire 접근

```blade
<div x-data>
    <span x-text="$wire.count"></span>              {{-- 프로퍼티 읽기 --}}
    <button @click="$wire.count++">증가</button>      {{-- 프로퍼티 쓰기 --}}
    <button @click="$wire.save()">저장</button>        {{-- 메서드 호출 --}}

    <button @click="
        let result = await $wire.calculate(5);
        alert(result);
    ">계산</button>   {{-- 결과 받기 (Promise) --}}

    <button @click="$wire.$set('filter', 'active', false)">필터</button>   {{-- 재렌더링 없이 설정 --}}
    <button @click="$wire.$dispatch('refresh')">새로고침</button>
    <button @click="$wire.$refresh()">갱신</button>
</div>
```

### `$wire.$entangle` — 양방향 연결

Livewire 프로퍼티와 Alpine 상태를 묶는다.

```php
new class extends Component { public bool $showModal = false; };
```

```blade
<div x-data="{ open: $wire.$entangle('showModal') }">
    <button @click="open = true">열기</button>
    <div x-show="open" @click.away="open = false">
        모달 내용
        <button @click="open = false">닫기</button>
    </div>
</div>

{{-- 서버 왕복을 지연시키려면 (권장) --}}
<div x-data="{ open: $wire.$entangle('showModal').live }">
```

## Laravel 구현

### 실전 — 드롭다운 + Livewire 검색

```blade
<div x-data="{ open: false }" @click.away="open = false" class="relative">
    <button @click="open = !open">{{ $selectedName ?: '선택하세요' }}</button>

    <div x-show="open" x-transition class="absolute mt-1 w-64 bg-white border rounded shadow-lg">
        {{-- 검색은 서버로 --}}
        <input type="text" wire:model.live.debounce.300ms="search" class="w-full border-b p-2">

        <div class="max-h-60 overflow-y-auto">
            @foreach ($this->options as $option)
                <button wire:key="opt-{{ $option->id }}"
                        wire:click="select({{ $option->id }})"
                        @click="open = false">
                    {{ $option->name }}
                </button>
            @endforeach
        </div>
    </div>
</div>
```

### 판단 기준 — Alpine vs Livewire

| 하려는 일 | 사용 |
|---|---|
| 드롭다운 열기/닫기 | Alpine |
| 탭 전환 (내용이 이미 로드됨) | Alpine |
| 애니메이션, 트랜지션 | Alpine |
| 클립보드 복사, 포커스 이동 | Alpine |
| 폼 입력값 실시간 글자수 카운트 | Alpine |
| DB 조회가 필요한 검색 | Livewire |
| 데이터 저장/삭제 | Livewire |
| 권한 확인이 필요한 동작 | Livewire (서버) |

## 주의사항 / 안티패턴

- 순수 UI 상태(드롭다운 열림 여부 등)까지 Livewire 프로퍼티로 만들면 사소한 토글에도 서버 왕복이 생겨 체감 속도가 떨어진다 — 위 판단 기준표를 기본 원칙으로 삼는다.
- `$wire.$entangle`은 기본적으로 값이 바뀔 때마다 즉시 서버로 동기화한다 — 빈번한 토글(예: 슬라이더 드래그)에는 `.live` 없이 쓰면 매 프레임 요청이 나갈 수 있으니 필요에 따라 `.live` 유무를 결정한다.
- NativePHP 모바일 환경에서는 터치 반응처럼 즉각적이어야 하는 UI에 Alpine을 우선 적용하는 것이 특히 중요하다 ([[Livewire and NativePHP]] 참고).

## 참고

- [[Livewire Overview]] — Livewire와 Alpine의 역할 분담이 필요한 이유(네트워크 왕복 트레이드오프)
- [[Livewire and NativePHP]] — 모바일에서 Alpine 우선 적용이 특히 유용한 경우
- [[Livewire Nested Components and Props]] — `$wire`와 컴포넌트 간 통신의 관계
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

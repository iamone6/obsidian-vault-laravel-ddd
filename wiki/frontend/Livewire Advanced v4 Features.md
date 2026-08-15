---
title: Livewire Advanced v4 Features
category: frontend
tags: [frontend, livewire, laravel, v4]
related: [[Livewire Nested Components and Props]], [[Livewire Computed Properties]], [[Livewire and NativePHP]]
---

# Livewire Advanced v4 Features

v4에서 추가된 컴포넌트 내 JavaScript, Islands, Lazy/Defer 로딩, 드래그 정렬, 뷰포트 진입 감지, 요청 분리.

## 핵심 개념

### 컴포넌트 안의 JavaScript — v4 `<script>` 직접 사용

view 기반 컴포넌트(SFC/MFC)에서는 `@script` 래퍼 없이 `<script>`를 쓸 수 있다. 이 스크립트는 별도 캐시 파일로 제공되고 `$wire`가 자동 바인딩된다.

```php
new class extends Component {
    public int $count = 0;
    public function increment() { $this->count++; }
};
```
```blade
<div>
    <span>{{ $count }}</span>
    <button wire:click="increment">+</button>
    <button wire:click="$js.showAlert">알림</button>
</div>

<script>
    // this === $wire
    this.$js.showAlert = () => { alert(`현재 값: ${this.count}`); }
    console.log('컴포넌트 로드됨');   // 컴포넌트 초기화 시 실행
</script>
```

> **v3 대비**: v3의 `@script <script> $js('showAlert', () => {...}) </script> @endscript`는 v4에서 `this.$js.showAlert = () => {...}`로 단순화됐다.

### `wire:ref` — 엘리먼트 참조 (v4)

```blade
<div wire:ref="modal" class="hidden">모달 내용</div>
<button wire:click="$js.scrollToModal">모달로 이동</button>
<script>
    this.$js.scrollToModal = () => { this.$refs.modal.scrollIntoView({ behavior: 'smooth' }); }
</script>
```

### `@assets` — 외부 라이브러리 1회 로드

```blade
@assets
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
@endassets

<div wire:ignore><canvas id="chart"></canvas></div>
<script>
    new Chart(document.getElementById('chart'), { /* ... */ });
</script>
```

## Laravel 구현

### Islands — 컴포넌트 안의 독립 영역

컴포넌트를 쪼개지 않고도 일부만 독립적으로 갱신할 수 있다.

```php
#[Computed]
public function expensiveStats()
{
    return Order::selectRaw('SUM(total) as revenue, COUNT(*) as cnt')->first();
}
```
```blade
<div>
    <input wire:model.live="search">

    {{-- 이 안쪽은 별도로 갱신됨. search 변경 시 재계산되지 않음 --}}
    @island(name: 'stats', lazy: true)
        <div>매출: {{ $this->expensiveStats->revenue }}</div>
    @endisland
</div>
```

Island만 갱신하기: `<button wire:click="loadMore" wire:island.append="stats">더 보기</button>`

**언제 쓰나**: 무거운 통계 위젯, 사이드바, 알림 카운터처럼 "페이지 일부인데 갱신 주기가 다른" 것들. [[Livewire Nested Components and Props]]에서 다룬 자식 컴포넌트를 만드는 것보다 가볍다.

### Lazy / Defer — 지연 로딩

```blade
<livewire:revenue-chart lazy />     {{-- 화면에 보일 때 로드 --}}
<livewire:revenue-chart defer />    {{-- 초기 페이지 로드 직후 로드 --}}
<livewire:revenue lazy.bundle />    {{-- 여러 개를 묶어서 한 요청으로 --}}
```

```php
use Livewire\Attributes\{Lazy, Defer};

#[Lazy]
class RevenueChart extends Component { }

#[Lazy(bundle: true)]
class RevenueChart extends Component { }
```

플레이스홀더 지정:

```blade
@placeholder
<div class="h-64 bg-gray-100 animate-pulse rounded"></div>
@endplaceholder
```

### `wire:sort` — 드래그 앤 드롭 정렬 (v4)

```blade
<ul wire:sort="updateOrder">
    @foreach ($this->tasks as $task)
        <li wire:sort:item="{{ $task->id }}" wire:key="task-{{ $task->id }}">{{ $task->title }}</li>
    @endforeach
</ul>
```
```php
public function updateOrder(array $order)
{
    foreach ($order as $position => $id) {
        Task::where('id', $id)->update(['position' => $position]);
    }
}
```

### `wire:intersect` — 뷰포트 진입 감지 (v4)

```blade
<div wire:intersect="loadMore">더 불러오기</div>
<div wire:intersect.once="trackView">조회수 기록 (1회만)</div>
<div wire:intersect:leave="pauseVideo">화면 밖으로 나가면</div>
<div wire:intersect.margin.200px="loadMore">200px 전에 미리</div>
```

### `#[Isolate]` — 요청 분리

기본적으로 같은 페이지의 여러 컴포넌트 요청은 하나로 묶인다. 무거운 컴포넌트를 분리하고 싶을 때 쓴다.

```php
use Livewire\Attributes\Isolate;

#[Isolate]
class HeavyWidget extends Component { }
```

### 요청 가로채기 (Interceptor, v4)

```blade
<script>
    this.$intercept('save', ({ onSuccess, onError }) => {
        onSuccess(({ payload }) => { console.log('저장 성공'); });
        onError(() => { alert('저장 실패'); });
    })
</script>
```

> **v3 대비**: v3의 `Livewire.hook('commit', ...)`, `Livewire.hook('request', ...)`는 v4에서 `interceptMessage`/`interceptRequest`로 대체됐다(구 API도 호환).

### `wire:stream` — 실시간 스트리밍

LLM 응답처럼 조금씩 흘려보내야 할 때 쓴다.

```php
public function ask()
{
    foreach ($this->llmStream() as $chunk) {
        $this->stream($chunk, el: '#answer');   // v4 시그니처
    }
}
```
```blade
<div wire:stream="answer" id="answer"></div>
```

## 주의사항 / 안티패턴

- Island 안의 computed는 바깥 프로퍼티가 바뀌어도 재계산되지 않는다 — 이게 Island의 목적이지만, 반대로 Island 안의 값을 최신으로 유지하려면 `wire:island.append` 등으로 명시적으로 갱신해야 한다.
- `#[Lazy]` 컴포넌트는 초기 렌더에서 플레이스홀더만 나가므로, SEO가 중요한 콘텐츠에는 부적합할 수 있다.
- 저사양 모바일 기기(NativePHP)에서는 Islands/Lazy가 초기 렌더링 부담을 줄이는 데 특히 유용하다 ([[Livewire and NativePHP]] 참고).

## 참고

- [[Livewire Nested Components and Props]] — Islands 대신 별도 컴포넌트로 쪼갤지 판단하는 기준
- [[Livewire Computed Properties]] — Island 안에서 쓰는 `#[Computed]`
- [[Livewire and NativePHP]] — 모바일 환경에서 Lazy/Defer가 유용한 이유
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

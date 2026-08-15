---
title: Livewire Events
category: frontend
tags: [frontend, livewire, laravel]
related: [[Livewire Nested Components and Props]], [[Livewire Computed Properties]], [[Laravel Events]]
---

# Livewire Events

컴포넌트들은 서로를 직접 참조할 수 없다. 대신 이벤트로 통신한다.

## 핵심 개념

### 발행

```php
$this->dispatch('post-created');
$this->dispatch('post-created', postId: $post->id, title: $post->title);  // 이름 있는 인자만 가능
$this->dispatch('refresh')->to('post-list');       // 특정 컴포넌트에게만
$this->dispatch('refresh')->to(PostList::class);
$this->dispatch('refresh')->self();                // 자기 자신에게만
```

Blade에서 직접: `<button wire:click="$dispatch('open-modal', { name: 'confirm-delete' })">삭제</button>`

### 수신

```php
use Livewire\Attributes\On;

new class extends Component {
    #[On('post-created')]
    public function refreshList($postId, $title)
    {
        // 파라미터명이 dispatch의 인자명과 일치해야 함
        $this->reset('search');
    }

    #[On('post-updated.{postId}')]   // 동적 이벤트명
    public function onUpdated() { /* ... */ }
};
```

v3 스타일(레거시)도 동작한다.

```php
protected $listeners = [
    'post-created' => 'refreshList',
    'post-deleted' => '$refresh',     // 재렌더링만
];
```

### JavaScript에서 주고받기

```blade
<script>
    Livewire.dispatch('post-created', { postId: 5 });
    Livewire.dispatchTo('post-list', 'refresh', {});
    Livewire.on('post-created', (event) => { console.log(event.postId); });
</script>
```

PHP → 브라우저 이벤트:

```php
$this->dispatch('notify', message: '저장되었습니다', type: 'success');
```

```blade
<div x-data @notify.window="$dispatch('toast', { message: $event.detail.message })"></div>
```

## Laravel 구현

### 실전 패턴 1 — 모달

```php
{{-- resources/views/components/⚡modal.blade.php --}}
new class extends Component {
    public bool $show = false;
    public string $name = '';

    #[On('open-modal')]
    public function open(string $name)
    {
        if ($this->name !== $name) return;
        $this->show = true;
    }

    #[On('close-modal')]
    public function close() { $this->show = false; }
};
```

```blade
<div>
    @if ($show)
        <div class="fixed inset-0 bg-black/50" wire:click.self="close">
            <div class="bg-white rounded-lg p-6">
                {{ $slot ?? '' }}
                <button wire:click="close">닫기</button>
            </div>
        </div>
    @endif
</div>
```

### 실전 패턴 2 — 목록 갱신

```php
// 생성 컴포넌트
public function save()
{
    $post = Post::create([...]);
    $this->reset();
    $this->dispatch('post-created', postId: $post->id);
}
```

```php
// 목록 컴포넌트
#[On('post-created')]
public function refresh()
{
    unset($this->posts);   // computed 캐시 무효화 → 자동으로 다시 조회 ([[Livewire Computed Properties]])
}

#[Computed]
public function posts() { return Post::latest()->get(); }
```

### Laravel Echo 연동 (브로드캐스트)

```php
use Livewire\Attributes\On;

#[On('echo:orders,OrderShipped')]                          // 공개 채널
public function onOrderShipped($event) { $this->refresh(); }

#[On('echo-private:orders.{orderId},OrderShipped')]         // 프라이빗 채널
public function onShipped($event) { /* ... */ }

#[On('echo-presence:room.{roomId},here')]                    // presence 채널
public function onHere($event) { /* ... */ }
```

Echo 브로드캐스트 자체의 채널 인가·이벤트 클래스 설계는 [[Laravel Events]] 참고.

## 주의사항 / 안티패턴

- `#[On]` 메서드의 파라미터명은 `dispatch`가 보낸 이름 있는 인자명과 **정확히 일치**해야 한다. 불일치하면 값이 전달되지 않는다.
- 이벤트를 받고도 [[Livewire Computed Properties]] 캐시를 `unset`하지 않으면 데이터는 갱신됐는데 화면은 그대로인 상황이 생긴다.
- `$dispatch('refresh')`처럼 대상을 지정하지 않으면 같은 이름을 수신하는 **모든** 컴포넌트가 반응한다 — 의도치 않은 컴포넌트까지 재렌더링될 수 있으므로 필요하면 `->to()`/`->self()`로 좁힌다.

## 참고

- [[Livewire Nested Components and Props]] — `$parent` 메서드 호출과의 차이
- [[Livewire Computed Properties]] — 이벤트 수신 후 캐시 무효화
- [[Laravel Events]] — Echo 브로드캐스트의 기반이 되는 Laravel 이벤트 시스템
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

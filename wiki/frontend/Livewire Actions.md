---
title: Livewire Actions
category: frontend
tags: [frontend, livewire, laravel]
related: [[Livewire Properties]], [[Livewire Loading States]], [[Policy and Gate]]
---

# Livewire Actions

프론트엔드에서 직접 호출할 수 있는 컴포넌트 메서드. 파라미터 보안, 이벤트 디렉티브, 매직 액션, 반환값 활용까지.

## 핵심 개념

```php
new class extends Component {
    public array $items = [];
    public function addItem() { $this->items[] = '새 항목'; }
};
```
```blade
<button wire:click="addItem">추가</button>
```

### 파라미터 전달과 보안

```blade
<button wire:click="remove({{ $post->id }})">삭제</button>
<button wire:click="setStatus({{ $post->id }}, 'published')">발행</button>
```

> ⚠️ **파라미터는 클라이언트에서 오는 값이다.** 사용자가 조작할 수 있다.

```php
// ❌ 위험 — 남의 글도 지워짐
public function delete(int $id) { Post::find($id)->delete(); }

// ✅ 권한 확인
public function delete(int $id)
{
    $post = Post::findOrFail($id);
    $this->authorize('delete', $post);
    $post->delete();
}
```

액션 파라미터에도 라우트 모델 바인딩이 동작한다.

```php
public function delete(Post $post)   // ID를 넘기면 자동으로 모델 조회
{
    $this->authorize('delete', $post);
    $post->delete();
}
```

### 이벤트 디렉티브

```blade
<button wire:click="save">클릭</button>
<form wire:submit="save">폼 제출</form>
<input wire:keydown.enter="search">
<div wire:mouseenter="showTooltip">호버</div>
{{-- 임의의 DOM 이벤트도 가능 --}}
<div wire:scroll="onScroll">
<video wire:ended="nextVideo">
```

### 수식어(modifier)

```blade
<button wire:click.stop="edit">                     {{-- 이벤트 전파 막기 --}}
<form wire:submit.prevent="save">                    {{-- 기본 동작 막기 --}}
{{-- 참고: wire:submit은 v3부터 prevent가 기본 적용됨. 명시할 필요 없음 --}}
<div wire:click.self="close">                        {{-- 자기 자신에게서 발생한 이벤트만 --}}
<button wire:click.once="claim">                     {{-- 한 번만 --}}
<input wire:keydown.debounce.500ms="search">
<button wire:click="delete" wire:confirm="정말 삭제하시겠습니까?">삭제</button>
<button wire:click.renderless="trackClick">추적만</button>       {{-- 렌더링 건너뛰기 (v4) --}}
<button wire:click.preserve-scroll="loadMore">더 보기</button>   {{-- 스크롤 위치 보존 (v4) --}}
<button wire:click.async="logActivity">로그</button>              {{-- 비동기 병렬 실행 (v4) --}}
```

### 매직 액션

메서드를 만들지 않고 프론트엔드에서 바로 쓸 수 있다.

```blade
<button wire:click="$set('filter', 'active')">활성만</button>
<button wire:click="$toggle('showDetails')">상세 보기</button>
<button wire:click="$refresh">새로고침</button>
<button wire:click="$dispatch('post-created', { id: 5 })">알림</button>
<button wire:click="$parent.removeItem({{ $id }})">삭제</button>
```

## Laravel 구현

### 액션 반환값 활용

```php
public function save()
{
    Post::create([...]);

    return $this->redirect('/posts');
    return $this->redirect('/posts', navigate: true);       // SPA 방식 이동
    return $this->redirectRoute('posts.index', navigate: true);
    return $this->redirectIntended('/dashboard');
}

public function export()
{
    return response()->download(storage_path('app/report.pdf'));
}

public function exportCsv()
{
    return response()->streamDownload(function () {
        echo "id,name\n";
        foreach (Post::cursor() as $post) {
            echo "{$post->id},{$post->title}\n";
        }
    }, 'posts.csv');
}
```

### 플래시 메시지 / JavaScript 실행

```php
public function save()
{
    Post::create([...]);
    session()->flash('message', '저장되었습니다.');
    // 또는
    $this->dispatch('notify', message: '저장되었습니다.');

    $this->js("alert('저장 완료')");
    $this->js('$wire.showToast()');
}
```

### `#[Renderless]` — 재렌더링 건너뛰기

조회수 카운트처럼 화면을 바꿀 필요가 없는 액션에 쓴다.

```php
use Livewire\Attributes\Renderless;

#[Renderless]
public function incrementViewCount() { $this->post->increment('views'); }

// 또는 메서드 안에서
public function trackEvent() { $this->skipRender(); }
```

### `#[Async]` — 병렬 실행 (v4)

기본적으로 Livewire 요청은 순차 처리된다. 느린 액션이 다른 액션을 막는다. `#[Async]`를 붙이면 병렬로 돈다.

```php
use Livewire\Attributes\Async;

#[Async]
public function logActivity()
{
    ActivityLog::create([...]);   // 느려도 다른 조작을 막지 않음
}
```

## 주의사항 / 안티패턴

- 액션 파라미터(`int $id` 등)를 서버가 신뢰해서는 안 된다 — 라우트 모델 바인딩 + `authorize()`로 항상 권한을 재확인한다.
- 문자열 파라미터는 Blade 안에서 따옴표가 필요하다: `wire:click="setStatus({{ $post->id }}, 'published')"`.
- `#[Async]`는 순서 보장이 없는 작업(로그 기록 등)에만 쓴다 — 다른 액션과 순서에 의존하는 로직에는 부적합하다.

## 참고

- [[Livewire Properties]] — 액션이 변경하는 상태
- [[Livewire Loading States]] — `wire:target`으로 특정 액션의 로딩 상태만 표시
- [[Policy and Gate]] — `authorize()`가 위임하는 인가 로직
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

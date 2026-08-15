---
title: Livewire Properties
category: frontend
tags: [frontend, livewire, laravel, state]
related: [[Livewire Render Cycle]], [[Livewire Actions]], [[Livewire Computed Properties]]
---

# Livewire Properties

컴포넌트의 상태를 담는 public/protected 프로퍼티, `wire:model` 양방향 바인딩, 배열/Eloquent 바인딩, 프로퍼티 보호 어트리뷰트.

## 핵심 개념

```php
new class extends Component {
    public string $name = '홍길동';
    public int $age = 30;
    public array $tags = ['php', 'laravel'];
};
```

```blade
<p>{{ $name }} ({{ $age }}세)</p>  {{-- public은 $this 없이 바로 접근 --}}
<p>{{ $this->name }}</p>            {{-- 이렇게도 됨 --}}
```

`protected`/`private`는 `$this->`로만 접근하고 클라이언트로 나가지 않는다.

### `wire:model` — 양방향 바인딩

Livewire의 심장. **기본 동작은 "지연"**이다 — 타이핑해도 서버에 요청이 가지 않고, 다음 액션이 일어날 때 값이 함께 전송된다(글자마다 요청을 보내면 낭비이므로 의도된 설계).

```blade
<input type="text" wire:model="name">
```

#### 수식어(modifier)

```blade
<input wire:model.live="search">                    {{-- 타이핑마다 즉시 서버 요청 --}}
<input wire:model.live.debounce.500ms="search">      {{-- 입력 멈추고 500ms 후 --}}
<input wire:model.live.throttle.250ms="search">      {{-- 250ms마다 최대 1회 --}}
<input wire:model.blur="email">                      {{-- 포커스를 잃을 때 --}}
<select wire:model.change="category">                 {{-- change 이벤트 시 --}}
<input wire:model.blur.enter="search">                {{-- Enter 또는 blur 시 --}}
<input type="number" wire:model.number="quantity">    {{-- 숫자로 캐스팅 --}}
<input wire:model.trim="username">                    {{-- 앞뒤 공백 제거 --}}
<input wire:model.fill="nickname">                     {{-- 빈 문자열을 null로 --}}
```

> **v3 대비 — 중요한 변경**
> v3에서 `.blur`/`.change`는 **네트워크 요청 타이밍만** 제어했고, 클라이언트 상태(`$wire.property`)는 타이핑 즉시 갱신됐다.
> v4는 **클라이언트 상태 동기화 타이밍까지** 제어한다. v3와 동일하게 만들려면 `wire:model.live.blur="title"`을 쓴다. `.lazy`는 그대로 호환된다.

#### 중첩 프로퍼티 접근

```php
public array $form = [
    'title' => '',
    'author' => ['name' => '', 'email' => ''],
];
```

```blade
<input wire:model="form.title">
<input wire:model="form.author.name">
{{-- v4부터 대괄호 표기법도 지원 --}}
<input wire:model="form['author']['name']">
<input wire:model="items[0].name">
```

## Laravel 구현

### 배열 다루기

```php
new class extends Component {
    public array $todos = [];
    public string $newTodo = '';

    public function add()
    {
        $this->todos[] = $this->newTodo;
        $this->newTodo = '';
    }

    public function remove(int $index)
    {
        unset($this->todos[$index]);
        // ⚠️ 인덱스에 구멍이 생김. JSON 직렬화 시 객체가 되어버림
        $this->todos = array_values($this->todos);  // 반드시 재정렬
    }
};
```

### Eloquent 모델 바인딩

```php
new class extends Component {
    public Post $post;

    public function mount(Post $post)
    {
        $this->post = $post;
    }
};
```

Livewire는 모델을 통째로 직렬화하지 않고 **ID만 저장했다가 다음 요청에서 다시 조회**한다. 그래서 관계를 미리 로드해도(`$post->load('comments')`) 다음 요청에서 사라진다 — [[Livewire Computed Properties]]의 `#[Computed]`로 매번 다시 조회하거나 필요한 값만 스칼라로 뽑아둔다.

### 프로퍼티 관련 어트리뷰트

**`#[Locked]`** — 프론트엔드 조작 방지. **ID류 프로퍼티에는 습관적으로 붙인다.** 안 붙이면 사용자가 개발자도구로 `$wire.postId = 999` 해서 남의 글에 댓글을 달 수 있다.

```php
use Livewire\Attributes\Locked;

#[Locked]
public int $postId;      // 클라이언트가 값을 바꾸려 하면 예외 발생
```

**`#[Session]`** — 세션에 자동 저장, 페이지를 떠났다 돌아와도 유지.

```php
#[Session]
public string $sortBy = 'created_at';

#[Session(key: 'user-filter')]
public string $filter = 'all';
```

**`#[Url]`** — URL 쿼리스트링과 동기화 ([[Livewire URL and Navigation]] 참고).

```php
#[Url]
public string $search = '';   // ?search=laravel
```

### `reset()` / `fill()` / `pull()`

```php
$this->reset();                       // 모든 프로퍼티를 초기값으로
$this->reset('title', 'body');        // 특정 프로퍼티만
$this->resetExcept('search');         // 특정 것 제외하고 전부
$this->fill(['title' => '기본 제목']); // 여러 값 한 번에 세팅
$value = $this->pull('title');        // 값을 꺼내고 리셋
```

## 주의사항 / 안티패턴

- ID류나 권한 관련 프로퍼티에 `#[Locked]`를 빠뜨리면 클라이언트에서 값을 조작할 수 있다.
- 배열에서 `unset()` 후 `array_values()`로 재정렬하지 않으면 인덱스에 구멍이 생겨 JSON 직렬화 시 배열이 아니라 객체로 바뀐다.
- `wire:model`(수식어 없음)은 지연 바인딩이라는 걸 잊고 "왜 실시간으로 안 바뀌지" 하는 경우가 흔하다 — 즉시 반응이 필요하면 `.live`를 붙인다.

## 참고

- [[Livewire Render Cycle]] — public 프로퍼티가 왕복하는 이유, 직렬화 가능 타입
- [[Livewire Actions]] — 프로퍼티를 변경시키는 메서드
- [[Livewire Computed Properties]] — Eloquent 관계 재로드 문제의 해법
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

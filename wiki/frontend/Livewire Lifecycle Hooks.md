---
title: Livewire Lifecycle Hooks
category: frontend
tags: [frontend, livewire, laravel]
related: [[Livewire Render Cycle]], [[Livewire Properties]], [[Livewire Computed Properties]]
---

# Livewire Lifecycle Hooks

컴포넌트 생애 주기 곳곳에 끼어들 수 있는 훅. [[Livewire Render Cycle]]의 hydrate/dehydrate/render를 세분화한 것이다.

## 핵심 개념

### 전체 흐름

```
[최초 요청]
  boot()          → 매 요청 시작 시 (최초 포함)
  mount()         → 최초 1회만. 생성자 역할
  booted()        → boot + mount 이후
  rendering()     → render() 직전
  render()
  rendered()      → render() 이후
  dehydrate()     → 상태 직렬화 직전

[후속 요청]
  hydrate()       → 상태 복원 직후
  boot()
  booted()
  updating($name, $value)   → 프로퍼티 변경 직전
  updated($name, $value)    → 프로퍼티 변경 직후
  [액션 실행]
  rendering()
  render()
  rendered()
  dehydrate()
```

### `mount()` — 초기화

```php
new class extends Component {
    public Post $post;
    public string $title = '';

    public function mount(Post $post)
    {
        $this->post  = $post;
        $this->title = $post->title;
    }
};
```

> `mount()`는 **최초 1회만** 실행된다. 버튼을 눌러도 다시 실행되지 않는다. 컨트롤러의 생성자 + `index()` 초반 세팅을 합친 역할이다. 파라미터가 프로퍼티명과 같으면 `mount()`를 생략할 수 있다.

### `boot()` / `booted()` — 매 요청마다

`private`/`protected` 프로퍼티를 매 요청마다 다시 세팅할 때 쓴다.

```php
new class extends Component {
    public int $postId;
    protected Post $post;      // 직렬화 불가 → 매번 다시 세팅

    public function boot()
    {
        $this->post = Post::findOrFail($this->postId);
    }
};
```

의존성 주입도 된다: `public function boot(PostService $service) { $this->service = $service; }`

## Laravel 구현

### `updating()` / `updated()` — 프로퍼티 변경 감지

```php
new class extends Component {
    public string $search = '';

    // 모든 프로퍼티 대상
    public function updating(string $name, mixed $value) { logger("변경 예정: {$name} = {$value}"); }
    public function updated(string $name, mixed $value)
    {
        if (in_array($name, ['search', 'category'])) $this->resetPage();
    }

    // 특정 프로퍼티만 — updated + 프로퍼티명(PascalCase)
    public function updatedSearch(string $value) { $this->resetPage(); }
    public function updatingCategory(string $value) { /* 변경 전 값 검사 */ }
};
```

배열/중첩 프로퍼티의 경우:

```php
public array $form = ['title' => '', 'body' => ''];

public function updatedForm($value, $key) { /* $key = 'title' */ }        // form 전체
public function updatedFormTitle($value) { /* ... */ }                     // form.title 만
```

### `hydrate()` / `dehydrate()`

```php
public function hydrate() { /* snapshot에서 복원된 직후 */ }
public function dehydrate() { /* 직렬화 직전. 마지막 정리 작업 */ }
```

### `rendering()` / `rendered()`

```php
public function rendering($view, $data) { /* render() 직전. 뷰 데이터 조작 가능 */ }
public function rendered($view, $html) { /* 최종 HTML을 볼 수 있음. 디버깅에 유용 */ }
```

### `exception()` — 예외 가로채기

```php
public function exception($e, $stopPropagation)
{
    if ($e instanceof ModelNotFoundException) {
        $this->addError('search', '해당 항목을 찾을 수 없습니다.');
        $stopPropagation();   // 에러 페이지로 안 넘어감
    }
}
```

## 주의사항 / 안티패턴

- `mount()`에서 무거운 연산 결과를 `private`/`protected` 프로퍼티에 저장해도 다음 요청에서 사라진다 — `boot()`로 옮겨야 한다 ([[Livewire Render Cycle]] 결론 1 참고).
- `updated*` 콜백 이름은 프로퍼티명을 정확히 PascalCase로 붙여야 한다 (`search` → `updatedSearch`). 오타가 나면 조용히 호출되지 않는다.

## 참고

- [[Livewire Render Cycle]] — hydrate/dehydrate가 일어나는 이유
- [[Livewire Properties]] — 훅이 다루는 프로퍼티
- [[Livewire Computed Properties]] — `updated()`에서 흔히 하는 캐시 무효화 패턴
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

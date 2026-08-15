---
title: Livewire Render Cycle
category: frontend
tags: [frontend, livewire, laravel]
related: [[Livewire Overview]], [[Livewire Properties]], [[Livewire Lifecycle Hooks]]
---

# Livewire Render Cycle

Livewire의 요청-응답 사이클(snapshot, hydrate/dehydrate, morph)과 여기서 파생되는 4가지 핵심 결론. 이 장을 이해하면 이후 겪는 대부분의 버그가 스스로 설명된다.

## 핵심 개념

### 최초 렌더링

1. 브라우저가 페이지를 요청
2. Laravel이 Livewire 컴포넌트를 실행 → HTML 생성
3. 이 HTML에 **`snapshot`**(컴포넌트 현재 상태를 직렬화한 JSON)이 함께 붙어서 나감

```html
<div wire:id="abc123" wire:snapshot='{"data":{"search":"","count":0},"memo":{...},"checksum":"..."}'>
    <!-- 렌더링된 HTML -->
</div>
```

### 상호작용이 일어날 때

사용자가 버튼을 클릭하면 브라우저가 `snapshot`과 호출할 메서드를 담아 `POST /livewire-{hash}/update`를 보내고, 서버는 다음 순서로 처리한다.

1. snapshot으로 컴포넌트 재구성 (**hydrate**)
2. 요청된 메서드 실행 (예: `increment()`)
3. `render()` 다시 호출 → 새 HTML
4. 상태를 다시 직렬화 (**dehydrate**)
5. 브라우저가 `{ snapshot, html }`을 받아 기존 DOM과 새 HTML을 비교(**morph**) — 바뀐 부분만 갈아끼움

핵심 용어 3개:

- **Hydration(수화)**: JSON snapshot → PHP 객체 복원
- **Dehydration(탈수)**: PHP 객체 → JSON snapshot 직렬화
- **Morphing**: 기존 DOM과 새 HTML을 비교해 바뀐 부분만 교체 — 전체를 갈아끼우지 않으므로 input 포커스나 스크롤 위치가 유지됨

## Laravel 구현

### 결론 1 — 컴포넌트 인스턴스는 요청 사이에 살아있지 않다

```php
new class extends Component {
    public $count = 0;
    private $cache = [];   // ❌ 다음 요청에서 사라짐

    public function mount()
    {
        $this->cache = expensiveThing();  // 소용없음
    }
};
```

`public` 프로퍼티만 snapshot에 담겨 왕복한다. `private`/`protected`는 매 요청마다 초기값으로 리셋된다 — 매 요청 재세팅이 필요하면 [[Livewire Lifecycle Hooks]]의 `boot()`를 쓴다.

### 결론 2 — public 프로퍼티는 전부 클라이언트로 나간다

```php
public $user;              // 브라우저가 다 볼 수 있음
public $isAdmin = false;   // ❌ 사용자가 조작 가능!
protected $apiKey;         // 안전 (하지만 요청 간 유지 안 됨)
```

브라우저 개발자도구에서 `$wire.isAdmin = true` 하면 그냥 바뀐다. **권한 판단을 public 프로퍼티에 의존하면 안 된다.** 서버에서 매번 `Auth::user()->isAdmin()`으로 확인하거나 [[Livewire Properties]]의 `#[Locked]`를 쓴다.

### 결론 3 — public 프로퍼티에 담을 수 있는 타입이 제한적이다

JSON으로 직렬화 가능해야 한다.

```php
// ✅ 가능
public string $name;
public int $count;
public bool $flag;
public array $items;
public ?Post $post;                  // Eloquent 모델 (ID만 저장 후 재조회)
public Collection $posts;            // Eloquent 컬렉션
public Carbon $date;                 // DateTime 계열

// ❌ 불가능
public Closure $callback;            // 클로저
public PDO $connection;              // 리소스
public SomeRandomClass $service;     // 일반 객체 (Wireable 구현 없이는)
```

### 결론 4 — `render()`는 매 요청마다 실행된다

버튼 하나 눌러도 `render()`가 통째로 다시 돈다. 여기에 무거운 쿼리를 넣으면 매 클릭마다 그 쿼리가 돈다 → [[Livewire Computed Properties]]로 해결한다.

## 주의사항 / 안티패턴

- `private`/`protected` 프로퍼티에 무거운 초기화 결과를 담고 요청 간 유지될 거라 기대하지 말 것 — `mount()`가 아니라 `boot()`로 매 요청 재세팅해야 한다.
- ID류나 권한 관련 public 프로퍼티는 클라이언트가 임의로 값을 바꿀 수 있다는 전제로 설계할 것 — `#[Locked]`나 서버 측 재검증이 필요하다.
- Eloquent 컬렉션처럼 큰 데이터를 public 프로퍼티에 담으면 매 요청마다 전체가 JSON으로 왕복한다.

## 참고

- [[Livewire Overview]] — 전체 개념과 장단점
- [[Livewire Properties]] — `#[Locked]` 등 프로퍼티 보호 어트리뷰트
- [[Livewire Lifecycle Hooks]] — `boot()`/`mount()`의 실행 시점 차이
- [[Livewire Computed Properties]] — `render()` 재실행 비용 해결
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

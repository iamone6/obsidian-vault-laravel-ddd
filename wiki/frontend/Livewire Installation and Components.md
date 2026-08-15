---
title: Livewire Installation and Components
category: frontend
tags: [frontend, livewire, laravel, setup]
related: [[Livewire Overview]], [[Livewire Properties]], [[Blade Component Basics]]
---

# Livewire Installation and Components

설치, 첫 컴포넌트 만들기, v4의 3가지 컴포넌트 형식(SFC/MFC/Class)과 파일-이름 매핑 규칙.

## 핵심 개념

### 설치

```bash
# 새 프로젝트 (스타터 킷 선택 시 v4 자동 설치)
laravel new my-app

# 기존 프로젝트
composer require livewire/livewire
php artisan optimize:clear
```

직접 만든 레이아웃이라면 스크립트/스타일을 명시한다 (Laravel 11+ 기본 레이아웃은 자동 주입).

```blade
<head>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
    @livewireStyles   {{-- v3에서 필수. v4는 자동 주입되지만 명시해도 무방 --}}
</head>
<body>
    {{ $slot }}
    @livewireScripts
</body>
```

### 첫 컴포넌트

```bash
php artisan make:livewire counter
# → resources/views/components/⚡counter.blade.php
```

> **⚡ 이모지가 붙는 이유**: v4부터 파일명에 ⚡를 붙여 일반 Blade 파일과 구분한다. 거슬리면 `config/livewire.php`에서 `'make_command' => ['emoji' => false]`로 끈다.

```php
<?php
use Livewire\Component;

new class extends Component {
    public int $count = 0;

    public function increment() { $this->count++; }
    public function decrement() { $this->count--; }
    public function reset_() { $this->count = 0; }  // reset은 예약어라 사용 불가
};
?>

<div class="p-6 text-center">
    <h1 class="text-6xl font-bold">{{ $count }}</h1>
    <div class="mt-4 flex gap-2 justify-center">
        <button wire:click="decrement">−</button>
        <button wire:click="increment">+</button>
        <button wire:click="reset_">리셋</button>
    </div>
</div>
```

> ⚠️ `reset`은 Livewire 내장 메서드 이름이라 오버라이드하면 문제가 생긴다. 예약된 이름: `reset`, `mount`, `render`, `dispatch`, `validate`, `fill`, `all` 등.

### 화면에 띄우기

```blade
{{-- 기존 Blade 안에 삽입 --}}
<livewire:counter />
```

> ⚠️ **v4부터 태그를 반드시 닫아야 한다.** `<livewire:counter>`처럼 열어두면 뒤 내용을 slot으로 해석해서 렌더링이 깨진다.

```php
// 페이지 전체로 라우팅 (v4 권장 — SFC/MFC는 이 방식이 필수)
Route::livewire('/counter', 'counter');
```

## Laravel 구현

### 컴포넌트 3가지 형식

세 형식 모두 **컴포넌트 이름은 동일**하므로 나중에 자유롭게 전환할 수 있다.

**(1) Single-File Component — SFC (기본값)**: 로직과 뷰가 한 파일. 대부분의 경우 이걸 쓴다.

```bash
php artisan make:livewire post.create
# → resources/views/components/post/⚡create.blade.php
```

**(2) Multi-File Component — MFC**: 컴포넌트가 커지거나 JS/CSS/테스트가 붙을 때.

```bash
php artisan make:livewire post.create --mfc --js --css --test
```
```
resources/views/components/post/⚡create/
├── create.php          # PHP 클래스
├── create.blade.php    # Blade 템플릿
├── create.js / create.css / create.global.css / create.test.php  # 전부 선택
```

**(3) Class-based (v3 방식)**: v3 마이그레이션 중이거나 상속/트레이트 구조가 복잡할 때.

```bash
php artisan make:livewire CreatePost --class
```
```php
// app/Livewire/CreatePost.php
class CreatePost extends Component
{
    public string $title = '';
    public function save() { /* ... */ }
    public function render() { return view('livewire.create-post'); }
}
```

기본 형식을 바꾸려면 `config/livewire.php`에서 `'make_command' => ['type' => 'class']`.

형식 간 변환: `php artisan livewire:convert post.create --mfc`

### 파일 경로 ↔ 컴포넌트 이름 매핑

| 형식 | 파일 경로 | 컴포넌트 이름 |
|---|---|---|
| SFC | `resources/views/components/post/⚡create.blade.php` | `post.create` |
| MFC | `resources/views/components/post/⚡create/create.php` | `post.create` |
| Class | `app/Livewire/Post/Create.php` | `post.create` |
| SFC (네임스페이스) | `resources/views/pages/post/⚡create.blade.php` | `pages::post.create` |

페이지 전용 컴포넌트는 `pages::` 네임스페이스로 분리해 정리한다.

```bash
php artisan make:livewire pages::post.index
```
```php
use Livewire\Attributes\{Layout, Title};

#[Layout('layouts::app')]
#[Title('게시글 목록')]
new class extends Component { /* ... */ };
```

## 주의사항 / 안티패턴

- **템플릿에는 루트 엘리먼트가 정확히 하나** 있어야 한다. Livewire가 이 루트 엘리먼트에 `wire:id`를 붙여 컴포넌트를 추적하기 때문이다.

```blade
{{-- ❌ 루트가 둘 --}}
<h1>제목</h1>
<p>내용</p>
```

- `<livewire:counter>`처럼 v4에서 태그를 안 닫으면 뒤 내용이 slot으로 해석되어 렌더링이 깨진다.
- [[Blade Component Basics]]의 `<x-component>`와는 다르다 — Livewire 컴포넌트는 자체 상태(snapshot)와 서버 왕복을 가지지만, Blade 컴포넌트는 순수 뷰 조합이다.

## 참고

- [[Livewire Overview]] — 전체 개념
- [[Livewire Properties]] — `mount()`로 초기 프로퍼티 주입
- [[Blade Component Basics]] — 상태 없는 Blade 컴포넌트와의 차이
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

---
title: Livewire URL and Navigation
category: frontend
tags: [frontend, livewire, laravel]
related: [[Livewire Properties]], [[Livewire Pagination and File Uploads]], [[Livewire and NativePHP]]
---

# Livewire URL and Navigation

필터/검색 상태를 URL에 반영하는 `#[Url]`과, 페이지 전체 새로고침 없이 전환하는 `wire:navigate`.

## 핵심 개념

### `#[Url]` 옵션

```php
use Livewire\Attributes\Url;

new class extends Component {
    #[Url]
    public string $search = '';                    // ?search=laravel

    #[Url(as: 'q')]
    public string $query = '';                      // ?q=laravel

    #[Url(history: true)]
    public string $tab = 'general';                  // 브라우저 히스토리에 기록 → 뒤로가기 동작

    #[Url(keep: true)]
    public string $sort = 'latest';                   // 값이 기본값이어도 URL에 남김

    #[Url(except: '')]
    public string $filter = '';                        // 빈 문자열일 때는 URL에서 제거

    #[Url(nullable: true)]
    public ?string $category = null;
};
```

### 실전 — 탭 UI

```php
new class extends Component {
    #[Url(history: true)]
    public string $tab = 'profile';

    public function setTab(string $tab) { $this->tab = $tab; }
};
```

```blade
@foreach (['profile' => '프로필', 'security' => '보안'] as $key => $label)
    <button wire:click="setTab('{{ $key }}')"
            class="{{ $tab === $key ? 'border-b-2 border-blue-600' : '' }}">
        {{ $label }}
    </button>
@endforeach

@if ($tab === 'profile')
    <livewire:settings.profile />
@endif
```

## Laravel 구현

### `wire:navigate` — SPA처럼 만들기

일반 링크에 `wire:navigate`를 붙이면 페이지 전체를 새로 불러오지 않고 SPA처럼 전환된다. JS/CSS를 다시 파싱하지 않으므로 체감 속도가 크게 개선된다.

```blade
<a href="/posts" wire:navigate>게시글</a>
<a href="/posts" wire:navigate.hover>게시글</a>          {{-- 마우스를 올렸을 때 미리 로드 --}}
<a href="/login" wire:navigate.replace>로그인</a>        {{-- 히스토리 대체 --}}
```

```php
return $this->redirect('/posts', navigate: true);
return $this->redirectRoute('posts.show', ['post' => $post], navigate: true);
```

현재 페이지 표시:

```blade
<a href="/posts" wire:navigate wire:current="text-blue-600 font-bold">게시글</a>
<a href="/" wire:navigate wire:current.exact="font-bold">홈</a>       {{-- 정확히 일치할 때만 --}}
```

### 상태 유지 (`@persist`)

페이지가 바뀌어도 특정 엘리먼트를 유지한다. 오디오 플레이어, 사이드바 스크롤 등에 쓴다.

```blade
@persist('player')
    <audio src="{{ $currentSong }}" controls></audio>
@endpersist

@persist('sidebar')
    <div class="overflow-y-scroll h-screen" wire:navigate:scroll>
        <!-- 사이드바 메뉴 -->
    </div>
@endpersist
```

> **v3 대비**: v3에서는 `wire:scroll`이었지만 v4에서는 `wire:navigate:scroll`로 바뀌었다.

### 네비게이션 이벤트

```blade
<script>
    document.addEventListener('livewire:navigate', (e) => { console.log('이동 시작', e.detail.url); });
    document.addEventListener('livewire:navigated', () => { /* 서드파티 스크립트 재초기화 등 */ });
</script>
```

> ⚠️ `wire:navigate`를 쓰면 페이지가 실제로 새로고침되지 않으므로, `DOMContentLoaded`에 걸어둔 스크립트가 다시 실행되지 않는다. `livewire:navigated` 이벤트를 대신 쓴다.

## 주의사항 / 안티패턴

- `#[Url]`에 `except`를 지정하지 않으면 기본값 상태에서도 URL에 파라미터가 계속 붙어 지저분해진다.
- `wire:navigate`로 전환한 페이지에서 `DOMContentLoaded` 기반 초기화 스크립트가 실행되지 않는 문제는 이 위키의 다크모드 FOUC 방지 스크립트([[Tailwind Dark Mode]])와는 반대 방향의 흔한 실수다 — 여기서는 "다시 실행 안 됨"이 문제다.
- NativePHP 맥락에서는 URL 개념이 약하므로 이 장의 상당 부분이 우선순위가 낮다 ([[Livewire and NativePHP]] 참고).

## 참고

- [[Livewire Properties]] — `#[Url]`과 함께 자주 쓰는 프로퍼티 어트리뷰트
- [[Livewire Pagination and File Uploads]] — 페이지네이션과 `#[Url]` 조합
- [[Livewire and NativePHP]] — 모바일 환경에서 이 장의 우선순위
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

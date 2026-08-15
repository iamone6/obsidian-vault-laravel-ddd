---
title: Livewire Overview
category: frontend
tags: [frontend, livewire, laravel]
related: [[Livewire Render Cycle]], [[Livewire Installation and Components]], [[Livewire and NativePHP]], [[Blade Component Basics]]
---

# Livewire Overview

"PHP 클래스의 프로퍼티를 바꾸면 화면이 알아서 갱신된다" — Vue/React가 클라이언트에서 하는 일(상태 → 화면 자동 반영)을 **서버 사이드 PHP에서** 하는 프레임워크. 기준 버전 v4.

## 핵심 개념

기존 Laravel 방식으로 실시간 검색을 만들려면 컨트롤러 + Blade + (실시간을 원하면) JS + API 라우트, 총 3~4곳에 로직이 흩어진다. Livewire는 파일 하나로 끝낸다.

```php
new class extends Component {
    public string $search = '';

    public function render()
    {
        return $this->view([
            'posts' => Post::where('title', 'like', "%{$this->search}%")->get(),
        ]);
    }
};
```

```blade
<div>
    <input type="text" wire:model.live="search" placeholder="검색...">
    @foreach ($posts as $post)
        <article wire:key="{{ $post->id }}">{{ $post->title }}</article>
    @endforeach
</div>
```

JS도 API 라우트도 없이, 타이핑할 때마다 목록이 실시간으로 바뀐다.

### 장단점

| 항목 | 평가 |
|---|---|
| 개발 속도 | 화면 하나에 파일 하나. 컨트롤러/API/JS 왕복이 없어짐 |
| 학습 곡선 | Laravel/Blade를 알면 진입 장벽이 낮음 |
| 검증/인증 | Laravel의 `validate()`, `Auth`, `Gate`, `Policy`를 그대로 사용 |
| **네트워크 왕복** | **모든 상호작용이 HTTP 요청.** 지연이 있는 환경에서는 체감이 나쁨 |
| **상태 전송량** | public 프로퍼티가 매 요청마다 왕복. 큰 배열/컬렉션을 담으면 무거워짐 |
| 복잡한 UI | 드래그, 캔버스, 실시간 애니메이션 등은 Alpine.js/JS 병행 필요 |

**적합한 곳**: 관리자 페이지, CRUD, 대시보드, 폼 위주 화면, 사내 도구
**부적합한 곳**: 오프라인 우선 앱, 밀리초 단위 반응이 필요한 UI, 채팅/게임

## Laravel 구현

> **NativePHP 맥락에서는 "네트워크 왕복"이라는 최대 약점이 대부분 사라진다.** 앱이 기기 안에서 돌기 때문에 이 왕복이 실제로는 localhost 호출이라 지연이 사실상 0이다. 이것이 NativePHP에서 Livewire를 권장하는 핵심 이유다. 자세한 건 [[Livewire and NativePHP]] 참고.

## 주의사항 / 안티패턴

- Livewire를 배우기 전에 [[Livewire Render Cycle]](동작 원리)을 먼저 이해할 것 — 이 위키뿐 아니라 원본 소스도 "이 절을 건너뛰지 말라"고 강조한다. 여기서 나오는 결론들(인스턴스 비영속성, public 프로퍼티 노출, 직렬화 가능 타입 제한)이 이후 겪는 대부분의 버그를 설명한다.
- 밀리초 단위 반응이 필요한 UI(드래그 캔버스, 실시간 애니메이션)에는 부적합 — Alpine.js 병행이나 순수 JS를 고려한다.

## 참고

- [[Livewire Render Cycle]] — snapshot/hydrate/dehydrate/morph, 반드시 먼저 읽어야 하는 장
- [[Livewire Installation and Components]] — 설치와 첫 컴포넌트
- [[Livewire and NativePHP]] — NativePHP 맥락에서 Livewire가 특히 유리한 이유
- [[Blade Component Basics]] — Blade 컴포넌트와의 관계(Livewire는 상태를 가진 Blade 컴포넌트에 가깝다)
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

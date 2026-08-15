---
title: Index — Laravel Wiki
type: meta
---

# Index

Laravel + DDD 위키 페이지 카탈로그. 카테고리별로 정리되어 있다.

---

## DDD 핵심 개념 (`wiki/ddd-core/`)

| 페이지 | 요약 |
|--------|------|
| [[Design Philosophy]] | DDD 핵심 원칙 — 언어 우선, 패턴은 가이드라인 |
| [[Bounded Context]] | 모델의 경계와 컨텍스트 매핑 |
| [[Entity]] | 식별자를 가진 도메인 객체 |
| [[Value Object]] | 불변, 동등성 기반 객체 |
| [[Aggregate]] | 일관성 경계를 갖는 객체 클러스터 |
| [[Domain Event]] | 도메인에서 발생한 사실 |
| [[Repository]] | 집합체 저장/조회 추상화 |
| [[Application Service]] | 유스케이스 오케스트레이션 |
| [[Ubiquitous Language]] | 개발자와 도메인 전문가의 공통 언어 |

---

## Laravel 통합 (`wiki/laravel/`)

| 페이지 | 요약 |
|--------|------|
| [[Directory Structure]] | DDD 기반 Laravel 디렉토리 구성 |
| [[Eloquent and DDD]] | Eloquent와 DDD의 임피던스 불일치 해결 |
| [[Service Provider]] | 도메인 바인딩과 서비스 프로바이더 |
| [[Container]] | 의존성 주입 컨테이너와 오토와이어링, 퍼사드 |
| [[Container Binding Attributes]] | `#[Bind]`/`#[BindWhen]`으로 인터페이스-구현체 바인딩을 선언부로 옮기는 방법 |
| [[Laravel Events]] | Laravel 이벤트 시스템과 도메인 이벤트 연결 |
| [[Action Pattern]] | 단일 책임 액션 클래스 |
| [[DTO]] | Data Transfer Object 구현 |
| [[Form Request]] | 입력 유효성 검사 레이어 |
| [[Policy and Gate]] | Gate/Policy 기반 인가와 도메인 인가 규칙의 분리 |
| [[Custom Query Builder]] | newEloquentBuilder 기반 Repository 대안 |
| [[ViewModel]] | 화면/응답별 조회 데이터 캡슐화, 가벼운 CQRS의 Query 절반 |
| [[Custom Collection]] | 컬렉션 단위 집계/계산 로직 캡슐화 |
| [[Third-Party Service Integration]] | 외부 API를 미니 애플리케이션으로 캡슐화 |
| [[Eloquent Recipes]] | 자주 쓰는 Eloquent 트릭 모음 (관계, N+1, 팩토리) |
| [[API Design]] | JSON API, Spatie Query Builder, 버전 관리, 베스트 프랙티스 |
| [[Eloquent Model Attributes]] | PHP 8 attribute로 모델 설정을 선언하는 24개 클래스 (Table, Fillable, ScopedBy, UseEloquentBuilder 등) |
| [[Laravel MCP]] | AI 클라이언트에 Tool/Resource/Prompt를 노출하는 공식 MCP 패키지 |
| [[Laravel Data]] | spatie/laravel-data — Attribute 기반 검증으로 FormRequest를 대체하는 DTO 패키지 |

---

## 아키텍처 패턴 (`wiki/architecture/`)

| 페이지 | 요약 |
|--------|------|
| [[Layered Architecture]] | 도메인·애플리케이션·인프라·UI 레이어 |
| [[CQRS]] | 명령/조회 책임 분리 |

---

## 구현 패턴 (`wiki/patterns/`)

| 페이지 | 요약 |
|--------|------|
| [[Repository Implementation]] | Eloquent 기반 리포지토리 구현 |
| [[Mapper Pattern]] | Eloquent 모델 ↔ 도메인 엔티티 변환 |
| [[Unit of Work]] | 트랜잭션 일관성 관리 |
| [[Pipeline Pattern]] | 미들웨어 파이프라인 |
| [[State Pattern]] | 상태별 클래스 분리와 전이(Transition) 캡슐화 |

---

## 테스트 (`wiki/testing/`)

| 페이지 | 요약 |
|--------|------|
| [[Domain Testing]] | 도메인 레이어 단위 테스트 |
| [[Feature Testing]] | Laravel Feature 테스트와 DDD |
| [[Testing Complex Features]] | 이벤트/큐/시간 의존 기능의 블랙박스 통합 테스트 |
| [[Test-Driven Development]] | Red-Green-Refactor 사이클 |

---

## 개발 도구 (`wiki/tooling/`)

| 페이지 | 요약 |
|--------|------|
| [[Static Analysis]] | PHPInsights, Larastan, Deptrac을 통한 품질·아키텍처 검사 |
| [[CI-CD Pipeline]] | Github Actions / Gitlab CI 파이프라인 구성 |
| [[Laravel Boost]] | AI 코딩 에이전트에 프로젝트 컨텍스트(스키마·라우트·컨벤션)를 제공하는 공식 MCP 도구 — Guidelines/Skills/MCP Server 3기둥 |

---

## 프론트엔드 (`wiki/frontend/`)

| 페이지 | 요약 |
|--------|------|
| [[Tailwind CSS]] | 유틸리티 우선 CSS 프레임워크 개요, 트레이드오프 (허브 페이지) |
| [[Tailwind Installation]] | Laravel+Vite 설치, 다른 빌드 환경, 에디터 설정 |
| [[Tailwind Utility Syntax]] | 클래스 이름 ↔ CSS 대응, 간격/색상 스케일, 임의 값 |
| [[Tailwind Layout]] | Flexbox, Grid, 컨테이너 쿼리 |
| [[Tailwind Variants]] | 상태/반응형/group·peer/data·aria variant |
| [[Tailwind Component Strategy]] | `@apply` 대신 Blade 컴포넌트로 반복 클래스 관리 |
| [[Tailwind Dynamic Class Pitfall]] | 동적 클래스 문자열 조합 함정과 해법 |
| [[Tailwind Theme Configuration]] | CSS-first 설정: `@theme`, `@utility`, `@custom-variant` |
| [[Tailwind Dark Mode]] | OS 추종/수동 토글/시맨틱 토큰 전략 |
| [[Tailwind Accessibility]] | 포커스, 색상 대비, 모션, 터치 타깃 체크리스트 |
| [[Tailwind Build and Performance]] | 빌드 흐름, 배포 체크리스트, 성능 특성 |
| [[Tailwind v3 to v4 Migration]] | 구조/네이밍/기본값 변화, 마이그레이션 도구 |
| [[Blade Template Inheritance]] | Controller→Child→Parent 렌더링 흐름, `@yield`/`@section`/`@show`/`@parent`/`@stack` |
| [[Blade Includes and Loops]] | `@include` 계열, `@each`, `@foreach`/`$loop`, `@once` |
| [[Blade Conditionals and Environment]] | `@if`/`@auth`/`@env`, `@csrf`/`@method`, `@json`, `@dd`/`@dump` |
| [[Blade Component Basics]] | `@props`, `$slot`/named slot, `@aware` |
| [[Blade Component Attributes]] | `$attributes`(ComponentAttributeBag), `:` prefix 타입 규칙 |
| [[Blade Style Directives]] | `@vite`, `@style`, `@class` |
| [[Livewire Overview]] | 개념, 기존 방식과 비교, 장단점 (허브 페이지) |
| [[Livewire Render Cycle]] | snapshot, hydrate/dehydrate/morph, 4가지 핵심 결론 |
| [[Livewire Installation and Components]] | 설치, SFC/MFC/Class 3형식, 파일-이름 매핑 |
| [[Livewire Properties]] | `wire:model`, 배열/Eloquent 바인딩, `#[Locked]`/`#[Session]`/`#[Url]` |
| [[Livewire Actions]] | 액션과 파라미터 보안, 매직 액션, `#[Renderless]`/`#[Async]` |
| [[Livewire Forms and Validation]] | `validate()`, `#[Validate]`, Form 객체 |
| [[Livewire Lifecycle Hooks]] | mount/boot/updating/updated/hydrate/dehydrate 전체 흐름 |
| [[Livewire Computed Properties]] | `#[Computed]` 메모이제이션과 캐시 옵션 |
| [[Livewire Rendering and wire-key]] | morphing과 `wire:key` 규칙, `wire:ignore`/`wire:poll` |
| [[Livewire Loading States]] | `wire:loading`/`wire:target`, `data-loading` |
| [[Livewire Events]] | `dispatch`/`#[On]`, 모달/목록갱신 패턴, Echo 연동 |
| [[Livewire Nested Components and Props]] | props 비반응형 함정, `#[Reactive]`/`#[Modelable]`, Slots |
| [[Livewire Pagination and File Uploads]] | WithPagination, 커서 페이지네이션, WithFileUploads |
| [[Livewire URL and Navigation]] | `#[Url]` 옵션, `wire:navigate`, `@persist` |
| [[Livewire Alpine Integration]] | `$wire`, `$wire.$entangle`, Alpine vs Livewire 판단기준 |
| [[Livewire Advanced v4 Features]] | 컴포넌트 내 JS, Islands, Lazy/Defer, `wire:sort`/`wire:intersect` |
| [[Livewire Testing]] | `Livewire::test()`, 주요 assertion |
| [[Livewire Common Pitfalls]] | 자주 겪는 함정 15개 모음 |
| [[Livewire and NativePHP]] | NativePHP v3/v4 맥락, 학습 우선순위, 모바일 고려사항 |
| [[NativePHP Mobile Overview]] | 큰 그림 3가지 핵심 개념, Laravel 웹 ↔ NativePHP 대응표 (허브 페이지) |
| [[NativePHP Mobile Environment Setup]] | 공통 요구사항, WSL 미지원/iOS Mac 필수 제약, iOS/Android 환경 구축 |
| [[NativePHP Mobile Installation]] | 설치 순서, `.env` 설정, `native:install`/`native:run`, `nativephp/` 디렉터리 |
| [[NativePHP SuperNative Architecture]] | 공유 메모리, NativeComponent, EDGE 컴포넌트, 웹뷰 마이그레이션 전략 |
| [[NativePHP Mobile Routing and Navigation]] | `Route::native()`, `navigate`/`back`/`replace`, `@navigate` 디렉티브 |
| [[NativePHP EDGE Components]] | `<native:*>` 컴포넌트 카테고리별 전체 목록 |
| [[NativePHP Core Plugins]] | Biometrics/Camera/Geolocation 등 코어 플러그인과 설치 절차 |
| [[NativePHP Mobile Testing]] | Interactions/Native Events/Navigation/Accessibility 테스트 영역 |
| [[NativePHP Mobile Deployment]] | Android/iOS 배포 절차, 버전 관리 env |
| [[NativePHP Mobile Practical Notes]] | 심화주제 지도, 실무 진행 순서, 라이선스/PHP버전/성숙도 주의사항 |

---

## 통계

- 전체 페이지: 89
- 최종 업데이트: 2026-08-12

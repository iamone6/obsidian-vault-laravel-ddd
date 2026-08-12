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

---

## 통계

- 전체 페이지: 60
- 최종 업데이트: 2026-08-12

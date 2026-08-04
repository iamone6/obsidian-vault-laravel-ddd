---
title: Layered Architecture
category: architecture
tags: [architecture, layers, ddd]
related: [[Directory Structure]], [[Bounded Context]]
---

# Layered Architecture

DDD에서 가장 기본적인 아키텍처. 4개 레이어로 코드를 분리한다. 의존성은 항상 아래 방향(도메인을 향해)으로만 흐른다.

## 레이어 구조

```
┌─────────────────────────────────┐
│         UI / Interface           │  HTTP, CLI, Queue
├─────────────────────────────────┤
│        Application Layer         │  Use Cases, Orchestration
├─────────────────────────────────┤
│          Domain Layer            │  Business Logic (Core)
├─────────────────────────────────┤
│       Infrastructure Layer       │  DB, Email, External APIs
└─────────────────────────────────┘
```

## 각 레이어 역할

### Domain Layer (핵심)
- [[Entity]], [[Value Object]], [[Aggregate]]
- [[Domain Event]]
- [[Repository]] 인터페이스
- Domain Service (특정 엔티티에 속하지 않는 도메인 로직 — 별도 페이지 예정)
- **의존성 없음**: 프레임워크, ORM, 외부 라이브러리에 의존하지 않는 순수 PHP

### Application Layer
- [[Application Service]] / [[Action Pattern]]
- Command, Query, DTO
- 트랜잭션 경계
- **Domain에만 의존**

### Infrastructure Layer
- Repository 구현체 (Eloquent)
- Eloquent 모델
- 메일, 알림, 외부 API 클라이언트
- **Domain + Application에 의존**

### UI / Interface Layer
- HTTP Controller, Form Request
- Artisan Command
- Job, Listener
- **Application에만 의존** (Domain은 직접 접근 안 함)

## 의존성 규칙

```
UI → Application → Domain ← Infrastructure
```

Infrastructure가 Domain 인터페이스를 구현하여 의존성 역전(DIP)이 적용된다.

```php
// Domain (인터페이스 정의)
interface OrderRepository { ... }

// Infrastructure (구현)
class EloquentOrderRepository implements OrderRepository { ... }

// Service Provider (바인딩)
$this->app->bind(OrderRepository::class, EloquentOrderRepository::class);
```

## Laravel에서의 매핑

| Laravel 구성요소 | 레이어 |
|----------------|--------|
| Controller | UI |
| Form Request | UI |
| Middleware | UI |
| Application Service / Action | Application |
| Job (비즈니스 로직) | Application |
| Eloquent Model (순수 데이터) | Infrastructure |
| Repository 구현체 | Infrastructure |
| 도메인 Entity/VO/Aggregate | Domain |
| Domain Event | Domain |

## 주의사항

- **레이어 스킵**: Controller가 Repository를 직접 호출하면 Application 레이어가 유명무실해진다.
- **Domain에서 Laravel Facade 사용**: `Auth::user()`, `Cache::get()` 같은 Facade를 Domain에서 사용하면 도메인이 Laravel에 결합된다.

## 참고

- [[Directory Structure]] — Laravel에서 레이어를 디렉토리로 표현
- [[Bounded Context]] — 레이어 구조를 감싸는 컨텍스트 경계
- [[Design Philosophy]] — 이 의존성 규칙이 지키려는 언어 우선 원칙

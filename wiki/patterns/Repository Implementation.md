---
title: Repository Implementation (Eloquent 연관관계와 쿼리)
category: patterns
tags: [laravel, eloquent, repository, query-builder, relationships]
related: [[Repository]], [[Mapper Pattern]], [[Eloquent and DDD]], [[Unit of Work]]
---

# Repository Implementation (Eloquent 연관관계와 쿼리)

[[Repository]] 구현체가 내부적으로 사용하는 Eloquent의 연관관계(relationship)와 쿼리 빌더 메커니즘. Repository 인터페이스/전체 구현 예시는 [[Repository]] 문서를 참고하고, 여기서는 그 구현이 의존하는 Eloquent 저수준 도구를 다룬다.

## 핵심 개념: Eloquent 연관관계 유형

관계형 데이터베이스의 연관 테이블을 다루기 위해 Eloquent가 제공하는 관계 유형:

| 관계 | 메서드 | 예시 |
|------|--------|------|
| 일대일 | `hasOne` / `belongsTo` | Contact ↔ PhoneNumber |
| 일대다 | `hasMany` / `belongsTo` | Customer → Orders |
| 다대다 | `belongsToMany` | User ↔ Role |
| 연결을 통한 다수 | `hasManyThrough` | Country → Posts (through Users) |
| 연결을 통한 단일 | `hasOneThrough` | Contact → Address (through 중개 모델) |
| 다형성 관계 | `morphTo` / `morphMany` | Favoritable (Contact, Event 등 여러 타입) |

```php
class Contact extends Model
{
    public function phoneNumber()
    {
        return $this->hasOne(PhoneNumber::class); // phone_numbers.contact_id 기본 가정
    }
}

class PhoneNumber extends Model
{
    public function contact()
    {
        return $this->belongsTo(Contact::class); // 역방향
    }
}
```

관계 메서드는 `$contact->phoneNumber` 처럼 **속성 접근 문법**으로도 호출된다 (내부적으로 매직 메서드가 관계로 인식해서 처리).

## Repository 구현에서의 활용

### Eager Loading으로 N+1 방지

```php
// N+1 발생: items를 순회할 때마다 별도 쿼리
OrderModel::all()->each(fn($o) => $o->items);

// Eager Loading: 연관관계를 한 번의 추가 쿼리로 미리 로드
OrderModel::with('items')->find($id->toString());
```

[[Repository]] 구현체의 `findById()`에서 필요한 연관관계를 항상 `with()`로 명시해 N+1 문제를 방지한다.

### 쿼리 빌더 체이닝

```php
OrderModel::query()
    ->where('customer_id', $customerId->toString())
    ->with('items')
    ->orderByDesc('created_at')
    ->paginate($perPage, page: $page);
```

Read Model(조회 전용 Repository)은 도메인 Entity로 변환하지 않고 이런 쿼리 빌더 체이닝 결과를 직접 반환해도 무방하다 — 조회는 도메인 불변식을 거치지 않기 때문이다.

### 트랜잭션과 결합

저장 시 연관 레코드를 함께 갱신해야 한다면 [[Unit of Work]]에서 다룬 `DB::transaction()`으로 감싼다.

## DDD 관점에서의 활용

- Eloquent의 관계 메서드(`hasMany`, `belongsToMany` 등)는 **인프라 레이어(Repository 구현체, Eloquent 모델)** 안에만 존재해야 한다. 도메인 Entity는 이런 관계를 알지 못하고, [[Mapper Pattern]]을 통해 이미 로드된 데이터를 값/컬렉션으로 받는다.
- 다형성 관계(`morphTo`)처럼 여러 타입을 넘나드는 관계는 도메인 모델링 시 별도의 Value Object나 인터페이스로 추상화하는 것이 Eloquent 구조 노출을 막는 데 도움이 된다.

## 더 많은 관계/쿼리 레시피

`hasOne`/`belongsTo`/`hasMany`/`belongsToMany`/`hasManyThrough` 외에도 Eloquent는 반복되는 조회 패턴을 관계나 헬퍼로 대체할 수 있는 도구를 여럿 제공한다 — `oldestOfMany`/`latestOfMany`/`ofMany`(최오래/최신/특정 기준 단일 레코드 관계), `whereRelation`/`whereBelongsTo`(join 대신 관계 기반 필터), `withDefault`(nullable 관계의 Null Object). 자세한 예제는 [[Eloquent Recipes]] 참고.

API Resource 레이어에서는 이 문서의 Eager Loading 원칙이 `whenLoaded()`([[Eloquent Recipes]])로 이어진다 — Repository가 `with()`로 미리 로드하지 않은 관계를 Resource가 직접 참조하면 N+1이 재발한다.

## 주의사항 / 안티패턴

- 도메인 Entity가 Eloquent 관계 메서드를 직접 호출하게 만들지 말 것 — Entity는 Eloquent를 몰라야 한다.
- Lazy Loading을 그대로 둔 채 목록을 순회하면 N+1 쿼리가 발생한다. 항상 필요한 관계를 `with()`로 미리 선언한다.
- 관계 컬럼명이 컨벤션과 다르면(`contact_id`가 아닌 `owner_id` 등) 관계 정의 시 명시적으로 전달해야 한다: `hasOne(PhoneNumber::class, 'owner_id')`.

## 참고

- [[Repository]] — 인터페이스와 전체 구현 예시
- [[Mapper Pattern]] — Eloquent 모델(관계 포함) ↔ 도메인 Entity 변환
- [[Unit of Work]] — 연관 레코드 저장을 트랜잭션으로 묶기
- [[Eloquent Recipes]] — 관계/쿼리 레시피 모음
- 소스: 처음부터 제대로 배우는 라라벨, 5장 데이터베이스와 엘로퀀트

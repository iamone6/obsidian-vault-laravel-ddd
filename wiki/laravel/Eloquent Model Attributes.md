---
title: Eloquent Model Attributes
category: laravel
tags: [laravel, eloquent, attributes, php8, model-configuration, reflection]
related: [[Eloquent and DDD]], [[Eloquent Recipes]], [[Custom Query Builder]], [[Custom Collection]], [[Policy and Gate]]
---

# Eloquent Model Attributes

Laravel 10.44부터 13.x에 걸쳐 도입된 `Illuminate\Database\Eloquent\Attributes\*` 네임스페이스(총 24개, 13.22 기준). 모델의 **설정(configuration)**을 protected 프로퍼티나 메서드 오버라이드 대신 클래스 선언부의 PHP native attribute로 표현하는 방식이다.

## 핵심 개념

- **동작이 아니라 표기 방식의 문제다.** `#[Fillable(['name'])]`과 `protected $fillable = ['name'];`은 동일한 결과를 만든다. 새로운 기능이 생기는 게 아니라 기존 설정을 어디에 쓰느냐가 바뀔 뿐이다.
- **완전히 하위 호환이다.** 프로퍼티/메서드 오버라이드 방식은 deprecated되지 않았고, attribute를 전혀 안 써도 무방하다.
- **대부분 "프로퍼티가 기본값일 때만" attribute가 적용된다.** 단, `#[Table(name:)]`처럼 프로퍼티를 직접 선언하지 않은 경우 attribute가 이기는 예외도 있다 — 적용 조건은 attribute마다 다르므로 개별 확인이 필요하다.

도입 시점(요약):

```
v10.44  ObservedBy, ScopedBy
v11.28  CollectedBy
v11.39  UseFactory
v12.4   Scope (메서드 레벨)
v12.18  UsePolicy
v12.19  UseEloquentBuilder
v12.22  Boot, Initialize (메서드 레벨, 주로 trait용)
v12.29  UseResource, UseResourceCollection
v13.0   Appends, Connection, Fillable, Guarded, Hidden, Table, Touches, Unguarded, Visible
v13.2   DateFormat, WithoutIncrementing, WithoutTimestamps
v13.21  RouteKey
```

## 전체 목록

| Attribute | 대체하는 기존 방식 | 적용 조건 |
|---|---|---|
| `#[Table(name:, key:, keyType:, incrementing:, timestamps:, dateFormat:)]` | `$table`, `$primaryKey`, `$keyType`, `$incrementing`, `$timestamps`, `$dateFormat` | `$table`을 직접 선언하지 않았다면 attribute가 이김 (예외적) |
| `#[Connection]` | `$connection` | 프로퍼티가 `null`일 때만 |
| `#[WithoutIncrementing]` | `$incrementing = false` | 마커, `#[Table(incrementing:)]`보다 우선 |
| `#[WithoutTimestamps]` | `$timestamps = false` | 마커, `#[Table(timestamps:)]`보다 우선 |
| `#[DateFormat]` | `$dateFormat` | `#[Table(dateFormat:)]`보다 우선 |
| `#[Fillable]` | `$fillable` | **병합** (덮어쓰기 아님) |
| `#[Guarded]` / `#[Unguarded]` | `$guarded` | `$guarded === ['*']`(기본값)일 때만, `Unguarded`가 `Guarded`보다 우선 |
| `#[Hidden]` / `#[Visible]` | `$hidden` / `$visible` | **병합** |
| `#[Appends]` | `$appends` | **병합** |
| `#[Touches]` | `$touches` | `$touches`가 비어 있을 때만 |
| `#[ObservedBy]` | `Model::observe()` 등록 | 반복 가능(`IS_REPEATABLE`), 부모 체인 누적 병합 |
| `#[ScopedBy]` | `addGlobalScope()` in `booted()` | trait도 탐색, `IS_INSTANCEOF`로 커스텀 확장 가능 |
| `#[RouteKey]` | `getRouteKeyName()` 오버라이드 | 없으면 기본키로 폴백 |
| `#[UsePolicy]` | `$policies` 배열 / 네이밍 추론 | 상속 탐색 없음 |
| `#[UseResource]` / `#[UseResourceCollection]` | `toResource()`/`toResourceCollection()` 네이밍 추론 | 상속 탐색 없음 |
| `#[CollectedBy]` | `newCollection()` 오버라이드 | 부모 클래스 체인, trait은 미탐색 |
| `#[UseEloquentBuilder]` | `newEloquentBuilder()` 오버라이드 | 상속 탐색 없음 |
| `#[UseFactory]` | `newFactory()` 오버라이드 | 상속 탐색 없음 |
| `#[Scope]` (메서드 레벨) | `scopeXxx()` 네이밍 규칙 | `protected`/`public` 메서드만 (private 불가) |
| `#[Boot]` / `#[Initialize]` (메서드 레벨) | trait의 `bootTraitName()`/`initializeTraitName()` | 기존 네이밍 규칙과 병행 동작 |

## Laravel 구현

### 레거시 스키마 모델 예시

```php
use Illuminate\Database\Eloquent\Attributes\Table;
use Illuminate\Database\Eloquent\Attributes\WithoutIncrementing;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Attributes\Hidden;
use Illuminate\Database\Eloquent\Attributes\RouteKey;

#[Table(name: 'tb_article', key: 'article_uid', keyType: 'string', dateFormat: 'Y-m-d H:i:s')]
#[WithoutIncrementing]
#[RouteKey('slug')]
#[Fillable(['title', 'body', 'desk_code', 'published_at'])]
#[Hidden(['internal_memo'])]
class Article extends Model
{
    #[Scope]
    protected function byDesk(Builder $query, string $desk): void
    {
        $query->where('desk_code', $desk);
    }
}
```

컬럼/테이블 네이밍이 Laravel 관례와 다른 레거시 DB를 다룰 때, 프로퍼티 6개 이상이 붙던 클래스 상단이 하나의 선언부 블록으로 정리된다.

### `#[Scope]` — `scopeXxx` 네이밍 없이 로컬 스코프 선언

```php
class Post extends Model
{
    #[Scope]
    protected function published(Builder $query): void
    {
        $query->whereNotNull('published_at');
    }
}

Post::published()->get(); // 호출 방식은 기존과 동일
```

### `#[Boot]` / `#[Initialize]` — trait 이름 종속성 제거

**Before**: trait 이름을 바꾸면 `bootTraitName()`도 같이 바꿔야 했고, 안 바꾸면 조용히 호출되지 않았다.

```php
trait HasTenant
{
    #[Boot]
    public static function bootTenantScope(): void
    {
        static::addGlobalScope(new TenantScope);
    }

    #[Initialize]
    public function setUpTenantDefaults(): void
    {
        $this->mergeFillable(['tenant_id']);
    }
}
```

`#[Boot]`는 static 메서드에만, 부팅 시 한 번 호출. `#[Initialize]`는 인스턴스 메서드, 모델을 `new` 할 때마다 호출된다. 프레임워크 내부(`GuardsAttributes`, `HidesAttributes`, `HasTimestamps`)도 이미 이 방식을 쓴다.

## 공통 내부 동작 — `resolveClassAttribute()`

클래스 레벨 attribute 대부분이 이 헬퍼 하나를 통과한다(`ObservedBy`/`ScopedBy` 등 일부 예외 제외).

- **탐색 순서**: 자기 클래스 → 자기 클래스의 trait들 → 부모 클래스 → 부모의 trait들 → 최상위까지.
- **첫 번째 발견 하나만 사용**한다. 상속 체인에서 가장 가까운 것이 이긴다.
- **정적 캐싱**: `클래스@attribute@프로퍼티` 키로 프로세스 수명 동안 캐시된다. Octane(FrankenPHP/Swoole)처럼 워커가 오래 사는 환경에서도 리플렉션 비용은 첫 회 한 번뿐이지만, 런타임에 동적으로 값을 바꿀 여지도 없다.
- **예외를 삼킨다.** 리플렉션 실패 시 조용히 `null`이 되므로 attribute 클래스명 오타 등은 예외로 알려주지 않는다.

### trait/상속 탐색 지원 여부가 attribute마다 다르다

| 해석 경로 | trait 탐색 | 상속 처리 |
|---|---|---|
| `resolveClassAttribute()` (Table, Connection, Fillable, RouteKey 등 대부분) | O | 첫 번째 발견 하나 |
| `ScopedBy` | O | 부모 체인 누적 병합 |
| `ObservedBy` | X | 부모 체인 누적 병합 |
| `CollectedBy` | X | 부모 클래스 체인, 첫 번째 발견 |
| `UseEloquentBuilder`, `UseFactory`, `UsePolicy`, `UseResource(Collection)` | X | 해당 클래스만 — **상속 탐색 없음** |

`UseEloquentBuilder`/`UseFactory`/`UsePolicy`/`UseResource` 계열을 추상 베이스 모델에 붙여 자식이 물려받게 하는 설계는 동작하지 않는다 — 각 구체 클래스에 개별 부착해야 한다.

## DDD 관점에서의 활용

- `#[ScopedBy]`는 trait에 붙은 것도 수집하고 `IS_INSTANCEOF`로 조회하므로, Bounded Context마다 공통 스코프(예: 멀티테넌시)를 강제하는 커스텀 attribute를 만들 수 있다.

  ```php
  #[Attribute(Attribute::TARGET_CLASS | Attribute::IS_REPEATABLE)]
  class TenantScoped extends ScopedBy
  {
      public function __construct() { parent::__construct(TenantScope::class); }
  }

  #[TenantScoped]
  class Invoice extends Model {}
  ```

- [[Custom Query Builder]](`#[UseEloquentBuilder]`), [[Custom Collection]](`#[CollectedBy]`), [[Policy and Gate]](`#[UsePolicy]`)가 이미 다루는 "모델에 커스텀 클래스를 연결하는" 패턴들은 모두 이 attribute 계열로 선언부에 옮길 수 있다 — 각 문서의 "참고" 섹션에 교차 링크되어 있다.
- `#[Unguarded]`(대량 할당 방어를 끔)는 외부 입력이 직접 닿는 모델에는 쓰지 않는다 — 시딩/임포트 전용 모델로 한정하는 편이 [[Eloquent and DDD]]가 강조하는 "얇고 안전한 모델" 원칙에 부합한다.

## 주의사항 / 안티패턴

- **존재하지 않는 attribute를 쓰지 말 것**: `#[PrimaryKey]`, `#[KeyType]` 같은 이름은 13.22 소스에 없다. 기본키 관련 설정은 `#[Table(key:, keyType:)]`로 한다.
- **프로퍼티와 attribute를 한 모델에 섞지 말 것**: 병합형(`Fillable`, `Hidden`, `Visible`, `Appends`)과 "기본값일 때만 적용"형(`Guarded`, `Touches`, `connection`, `timestamps`, `dateFormat`)이 섞이면 "attribute를 고쳤는데 왜 안 바뀌지?" 하는 상황이 생긴다. 모델 단위로 한 가지 스타일을 정한다.
- **`Illuminate\Database\Eloquent\Casts\Attribute`와 이름이 겹친다.** 접근자/변경자 정의용 `Attribute`(→ [[Eloquent Recipes]])와 이 문서가 다루는 설정용 `Attributes\*`는 완전히 다른 클래스다. `use ... as`로 별칭을 주는 편이 안전하다.
- **`#[Scope]`는 `private` 메서드에 적용되지 않는다** — `protected` 또는 `public`이어야 한다.
- 조건부 로직이 필요한 설정(예: 환경에 따라 다른 테이블명)은 attribute로 표현할 수 없다 — attribute는 상수 표현식만 받으므로 이런 경우 프로퍼티/메서드 오버라이드 방식을 유지한다.

## 참고

- [[Eloquent and DDD]] — Eloquent 모델을 얇게 유지하는 원칙과 attribute 선언부의 관계
- [[Eloquent Recipes]] — `Casts\Attribute`(접근자/변경자)와의 이름 혼동 주의
- [[Custom Query Builder]] — `#[UseEloquentBuilder]`가 대체하는 `newEloquentBuilder()` 패턴
- [[Custom Collection]] — `#[CollectedBy]`가 대체하는 `newCollection()` 패턴
- [[Policy and Gate]] — `#[UsePolicy]`가 대체하는 정책 등록 패턴
- 소스: Laravel 13.22 기준 Eloquent Model Attribute 총정리

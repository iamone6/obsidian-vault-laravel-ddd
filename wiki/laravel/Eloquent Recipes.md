---
title: Eloquent Recipes
category: laravel
tags: [laravel, eloquent, recipes, n+1, relationships]
related: [[Repository Implementation]], [[Custom Query Builder]], [[Custom Collection]], [[Eloquent Model Attributes]]
---

# Eloquent Recipes

자주 재발명하기 쉬운 문제들을 Eloquent가 이미 제공하는 기능으로 해결하는 실전 레시피 모음. 개별 트릭이지만 여러 개가 [[Repository Implementation]]·[[Custom Query Builder]]의 실제 구현 품질을 좌우한다.

## 기본 Eloquent

### Attribute 캐스트 — 접근자/변경자를 한 메서드로

Laravel 8+ 방식. `getXAttribute`/`setXAttribute` 두 메서드 대신 하나로 통합한다.

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

class User extends Model
{
    protected function name(): Attribute
    {
        return new Attribute(
            get: fn (string $value) => Str::upper($value),
            set: fn (string $value) => Str::lower($value),
        );
    }
}
```

> 이름 혼동 주의: 여기서 쓰는 `Illuminate\Database\Eloquent\Casts\Attribute`(접근자/변경자)와 `Illuminate\Database\Eloquent\Attributes\*`(모델 설정용 PHP 8 attribute, 예: `#[Fillable]`, `#[Table]`)는 완전히 다른 클래스다. 자세한 내용은 [[Eloquent Model Attributes]] 참고.

### 모델 기본값 + 마이그레이션 기본값을 함께 쓰기

마이그레이션의 `default()`는 DB에 저장된 이후에만 적용된다. `new Order()`처럼 아직 저장하지 않은 인스턴스에서 기본값이 필요하면 모델에도 정의해야 한다.

```php
class Order extends Model
{
    protected $attributes = [
        'status' => App\Enums\OrderStatuses::DRAFT,
    ];
}
```

### invisible 컬럼 (MySQL 8+)

`select *`에도 절대 포함되지 않는 컬럼. 비밀번호, 토큰, 결제 정보에 적합하다.

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('password')->invisible();
});

$user = User::first();
$user->password; // null — 명시적으로 select('password')해야 값이 나옴
```

### 기타 단축 기법

- `saveQuietly()` — 모델 이벤트를 발생시키지 않고 저장
- `find([$id1, $id2])` — ID 배열로 여러 모델 한 번에 조회
- `isDirty()` / `getDirty()` — 아직 저장되지 않은 변경분 확인
- `push()` — 모델과 그 관계까지 한 번에 저장
- `updateOrCreate($match, $values)` — 조건에 맞으면 업데이트, 없으면 생성
- `when($condition, $callback)` — 조건부 쿼리 체이닝을 if문 없이
- `$appends` — 접근자를 배열/JSON 직렬화에 포함

## 관계(Relationship) 레시피

### whereRelation / whereBelongsTo — join 대신 관계로 필터링

```php
// join 대신
Holding::whereRelation('stock', 'ticker', 'AAPL')->get();

// where('user_id', $user->id) 대신
Order::whereBelongsTo($user)->sum('total_amount');
```

`whereRelation`은 내부적으로 `EXISTS` 서브쿼리를 쓰므로, 관계 테이블의 실제 컬럼 값이 필요하면(단순 필터링이 아니라) `join`이 낫다.

### oldestOfMany / latestOfMany / ofMany — "가장 오래된/최신 N개" 관계

매번 커스텀 쿼리를 작성하는 대신 관계 자체로 정의한다.

```php
class Employee extends Model
{
    public function oldestPaycheck() { return $this->hasOne(Paycheck::class)->oldestOfMany(); }
    public function latestPaycheck() { return $this->hasOne(Paycheck::class)->latestOfMany(); }
}

// 임의의 컬럼 기준
class User extends Authenticatable
{
    public function mostPopularPost()
    {
        return $this->hasOne(Post::class)->ofMany('like_count', 'max');
    }
}
```

`oldestOfMany`/`latestOfMany`는 auto-increment ID 기준이므로 UUID를 PK로 쓰면 동작하지 않는다.

### hasManyThrough — 중간 모델을 건너뛰는 관계

`$department->employees->paychecks`처럼 두 단계를 매번 순회하는 대신, 관계 자체로 한 번에 정의한다.

```php
class Department extends Model
{
    public function paychecks(): HasManyThrough
    {
        return $this->hasManyThrough(Paycheck::class, Employee::class);
    }
}
// $department->paychecks; 로 바로 접근
```

3단계 이상 깊은 관계(`Country → User → Post → Comment`)는 표준 Eloquent로 표현할 수 없다 — `staudenmeir/eloquent-has-many-deep` 패키지의 `hasManyDeep()`을 쓰면 서브쿼리로 한 번에 조회한다.

### withDefault — nullable 관계의 Null Object

```php
class Post extends Model
{
    public function author(): BelongsTo
    {
        return $this->belongsTo(User::class)->withDefault(['name' => 'Guest Author']);
    }
}
// $post->author?->name ?? 'Guest Author' 대신
$post->author->name; // 항상 안전
```

### withAvg — 관계의 평균값으로 정렬

```php
Book::query()->withAvg('ratings as average_rating', 'rating')->orderByDesc('average_rating');
```

### 관계 Eager Loading 시 필요한 컬럼만 로드

```php
Product::with('category:id,name')->get(); // select id, name from categories
```

### saveMany / createMany — 관계 벌크 저장

```php
$product->prices()->delete();
$product->prices()->saveMany($productPrices);   // 이미 만든 모델 컬렉션
$product->prices()->createMany($pricesArray);   // 배열에서 바로 생성
```

## API Resource와 N+1 — whenLoaded

[[Repository Implementation]]의 Eager Loading 원칙이 API Resource 레이어에서 특히 중요해지는 지점이다. Resource에서 관계를 직접 참조하면(`$this->department`), 관계가 미리 로드되지 않은 경우 레코드마다 추가 쿼리가 발생한다(500개 조회 시 501개 쿼리).

```php
class EmployeeResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->uuid,
            'department' => $this->whenLoaded('department'), // 로드 안 됐으면 쿼리 없이 null
        ];
    }
}

// 컨트롤러에서 명시적으로 eager load
$employees = Employee::with('department')->get();
```

## 마이그레이션 / 팩토리 레시피

```php
// 외래키 축약
$table->foreignId('category_id')->constrained();          // references('id')->on('categories')
$table->foreignIdFor(Category::class)->constrained();      // 모델 클래스에서 컬럼명까지 추론
$table->foreignId('category_id')->nullable()->constrained()->nullOnDelete();

// 팩토리: 관계 생성
$product = Product::factory()->for(Category::factory())->create();        // belongsTo
Category::factory()->has(Product::factory()->count(10));                   // hasMany
Product::factory()->has(ProductPrice::factory(), 'prices');                 // 관계명이 다를 때

// 팩토리: 상태(state)
class ProductFactory extends Factory
{
    public function inactive(): self
    {
        return $this->state(fn () => ['active' => false]);
    }
}
Product::factory()->state('inactive')->create();

// 팩토리: 생성 후 부수 작업
public function configure()
{
    return $this->afterCreating(function (User $user) {
        $user->profile_picture_path = /* 더미 이미지 생성 */;
        $user->save();
    });
}
```

## 기타

- **Prunable 트레이트**: 오래된/불필요한 레코드 삭제 로직을 표준화. `prunable(): Builder`로 대상 쿼리를 정의하면 `php artisan model:prune`이 스케줄러로 처리한다.
- **모든 쿼리 로깅**(로컬 환경): `AppServiceProvider::boot()`에서 `DB::listen()`으로 바인딩까지 포함한 SQL을 로그에 남긴다. 더 강력한 도구가 필요하면 Telescope, Clockwork, Laravel Debugbar.
- **pagination + query string**: `paginate()->withQueryString()`을 붙이지 않으면 필터/정렬 쿼리스트링이 페이지네이션 링크에서 사라진다.

## 주의사항 / 안티패턴

- API Resource에서 관계를 `whenLoaded()` 없이 직접 참조하지 말 것 — N+1의 가장 흔한 발생 지점이다.
- `oldestOfMany`/`latestOfMany`를 UUID 기반 관계에 사용하지 말 것 — auto-increment ID를 전제로 동작한다.
- invisible 컬럼은 보안 계층이 아니라 편의 기능이다 — 진짜 민감한 값(비밀번호 해시 등)은 여전히 암호화/해싱이 필요하다.

## 참고

- [[Repository Implementation]] — Eager Loading과 N+1 방지 원칙
- [[Custom Query Builder]] — `whereRelation` 등을 스코프로 추출하는 방법
- [[Custom Collection]] — 관계 결과를 집계하는 또 다른 방법
- [[Eloquent Model Attributes]] — `$fillable`/`$hidden`/`$appends` 등 프로퍼티 설정을 PHP 8 attribute로 옮기는 방법
- 소스: Laravel Eloquent Recipes (Martin Joo)

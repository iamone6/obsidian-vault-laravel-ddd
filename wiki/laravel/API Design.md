---
title: API Design (JSON API & Query Builder)
category: laravel
tags: [laravel, api, json-api, query-builder, versioning, rest]
related: [[Custom Query Builder]], [[Form Request]], [[DTO]], [[Eloquent Model Attributes]], [[Laravel MCP]]
---

# API Design (JSON API & Query Builder)

필터링·정렬·관계 포함·희소 필드셋(sparse fieldset)·버전 관리를 프로젝트마다 재발명하지 않고 표준화하는 방법.

## 핵심 개념: 왜 표준이 필요한가

API를 여러 개 만들다 보면 매번 다르게 설계하게 된다 — 관계를 응답에 포함하는 방식, camelCase/snake_case, 필터·정렬 문법이 프로젝트마다 제각각이다. **JSON API**는 이런 것들을 규격화한 명세다.

### JSON API 응답 구조

```json
{
  "data": {
    "id": 1,
    "type": "threads",
    "attributes": { "title": "This is a thread", "body": "..." },
    "relationships": {
      "author": { "data": { "type": "users", "id": 1 } },
      "comments": [{ "data": { "type": "comments", "id": 5 } }]
    }
  },
  "included": [
    { "id": 1, "type": "users", "attributes": { "name": "Micheal Scott" } }
  ]
}
```

- 모든 리소스는 최상위에 `id`와 `type`을 갖는다.
- 실제 속성은 `attributes` 객체 안에.
- `relationships`는 관계의 메타 정보(및 `self`/`related` 링크)만 담고, 실제로 로드된 관계 데이터는 `included`에 들어간다.
- 관계를 **전부 포함**하면 응답이 커지고 N+1이 생기기 쉽고, **전혀 포함하지 않으면** 클라이언트가 추가 요청을 N번 해야 한다 — JSON API는 `include` 쿼리 파라미터로 클라이언트가 직접 선택하게 해서 이 트레이드오프를 클라이언트에 위임한다.

> 명세를 100% 따라야 하는 것은 아니다. 마음에 안 드는 부분(예: relationships와 included의 분리)은 프로젝트에 맞게 변형해도 된다 — 중요한 것은 "회사/프로젝트 전체에서 일관된 기본값을 갖는 것"이다.

## Laravel 구현

### json-api 패키지 (Tim MacDonald)

`JsonResource` 대신 `JsonApiResource`를 상속하면 `id`/`type`/`attributes` 구조를 자동으로 만들어준다.

```php
use TiMacDonald\JsonApi\JsonApiResource;

class ThreadResource extends JsonApiResource
{
    public function toAttributes($request): array
    {
        return ['slug' => $this->slug, 'title' => $this->title];
    }

    public function toRelationships($request): array
    {
        return [
            'author' => fn () => new AuthorResource($this->author),
            'tags' => fn () => TagResource::collection($this->tags),
        ];
    }
}

// AppServiceProvider::boot() — id로 쓸 컬럼 지정 (auto-increment 대신 uuid 권장)
JsonApiResource::resolveIdUsing(fn (Model $model) => $model->uuid);
```

관계 클로저는 **지연 평가**된다 — 요청 URL에 `include=author`가 있을 때만 실제로 실행되므로, 클라이언트가 요청하지 않은 관계는 쿼리조차 발생하지 않는다.

### 요청 URL 문법 (필터/정렬/포함/필드)

```
GET /threads?filter[title]=laravel&filter[title]=php   # OR 조건 필터
GET /threads?sort=-like_count                           # -는 내림차순
GET /threads?sort=like_count,share_count                # 다중 정렬
GET /threads?include=tags,author                        # 관계 포함
GET /threads?fields[threads]=id,title,body               # 희소 필드셋 (select 최소화)
```

### Spatie laravel-query-builder — 위 규칙을 자동 구현

모델별로 어떤 필터/정렬/관계/필드를 허용할지만 선언하면, 요청 파라미터를 읽어 자동으로 쿼리를 만든다.

```php
$threads = QueryBuilder::for(Thread::class)
    ->allowedFilters(['title', 'body', 'author.name'])
    ->allowedSorts(['like_count', 'reply_count'])
    ->defaultSort('title')
    ->allowedIncludes(['author', 'tags'])
    ->allowedFields(['id', 'title'])
    ->where('category_id', $category->id)   // 커스텀 조건도 체이닝 가능
    ->get();
```

이 패키지가 [[Custom Query Builder]]의 스코프 위에서 필터/정렬/관계/필드셋을 선언적으로 조립해주는 상위 레이어라고 이해하면 된다.

### API 버전 관리

`RouteServiceProvider::boot()`에서 버전별 라우트 파일을 분리한다.

```php
public function boot()
{
    $this->routes(function () {
        Route::prefix('api/v1')
            ->middleware(['api', 'auth:sanctum'])
            ->group(base_path('routes/v1.php'));
    });
}
```

새 버전이 필요하면 `v2.php`를 추가하는 식으로 확장한다.

### 중첩 리소스 라우트

```php
Route::apiResource('categories', CategoryController::class)->only(['index', 'store', 'destroy']);
Route::apiResource('categories/{category}/threads', ThreadController::class)->except('update');
Route::apiResource('categories/{category}/threads/{thread}/replies', ReplyController::class)
    ->only(['index', 'store', 'destroy']);
```

`GET /api/v1/categories/my-category/threads`처럼 URL 자체가 리소스 계층을 드러내고, 각 컨트롤러는 표준 리소스 메서드 몇 개로 유지된다.

### `#[UseResource]` / `#[UseResourceCollection]` — 리소스 클래스를 모델에 직접 연결 (Laravel 12.29+)

json-api 패키지 없이 표준 `JsonResource`를 쓰는 경우, 모델과 리소스 클래스를 attribute로 연결하면 `guessResource()`의 네이밍 추론보다 우선 적용된다.

```php
use Illuminate\Database\Eloquent\Attributes\UseResource;
use Illuminate\Database\Eloquent\Attributes\UseResourceCollection;

#[UseResource(ArticleResource::class)]
#[UseResourceCollection(ArticleCollection::class)]
class Post extends Model {}

Post::find(1)->toResource();          // ArticleResource
Post::all()->toResourceCollection();  // ArticleCollection
```

모델명과 리소스명이 도메인 용어 차이로 불일치하는 경우 유용하다. 상속 탐색을 하지 않으므로 각 구체 모델에 개별 부착한다. 다른 모델 설정 attribute는 [[Eloquent Model Attributes]] 참고.

## API 베스트 프랙티스

- **auto-increment ID를 URL에 노출하지 않는다** — 공개 API라면 특히 보안 문제다. DB에는 정수 PK를 유지하되(MySQL 최적화), 모든 모델에 UUID를 추가로 두고 API에는 UUID만 노출한다.
- **HTTP 상태 코드를 제대로 쓴다** — 비동기(Job을 디스패치하는) 엔드포인트는 200이 아니라 `202 Accepted`와 상태 확인용 링크를 반환한다.
- **camelCase 응답** — JS 클라이언트 입장에서 자연스럽다.
- **클라이언트가 선택하게 한다** — 관계 포함, 필터, 정렬, 필드셋을 서버가 강제하지 않고 쿼리 파라미터로 위임한다.

## DDD 관점에서의 활용

- API Resource(`toAttributes`)는 [[DTO]]와 마찬가지로 도메인 객체를 외부 표현으로 변환하는 계층이다 — 도메인 Entity를 직접 노출하지 않는다는 원칙이 동일하게 적용된다.
- Query Builder의 필터/정렬 파라미터 검증은 [[Form Request]]의 `rules()`와 같은 층위에서 함께 다루는 것이 자연스럽다.

## 주의사항 / 안티패턴

- `include` 파라미터를 신뢰하고 무제한 관계를 허용하면, 클라이언트가 깊은 관계 체인을 요청해 N+1을 유발할 수 있다 — `allowedIncludes`로 화이트리스트를 명시한다.
- 관계를 "전부 포함 아니면 전혀 포함 안 함"으로 하드코딩하면 [[Eloquent Recipes]]가 다루는 `whenLoaded` N+1 문제를 그대로 안게 된다.

## 참고

- [[Custom Query Builder]] — Spatie QueryBuilder가 조립하는 기반 스코프
- [[Eloquent Recipes]] — `whenLoaded`를 통한 N+1 방지
- [[Form Request]] — 필터/정렬 파라미터 검증과의 접점
- [[Test-Driven Development]] — 이 저자가 API 개발에도 적용하는 개발 방법론
- [[Eloquent Model Attributes]] — `#[UseResource]`/`#[UseResourceCollection]`를 포함한 모델 설정 attribute 전체 목록
- [[Laravel MCP]] — 같은 유스케이스를 REST 클라이언트 대신 AI 클라이언트에 노출하는 대응 계층
- 소스: Proper API Design With Laravel (Martin Joo)

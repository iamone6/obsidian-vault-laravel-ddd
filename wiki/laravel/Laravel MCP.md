---
title: Laravel MCP
category: laravel
tags: [laravel, mcp, ai, model-context-protocol, tool-server]
related: [[Service Provider]], [[Container]], [[Form Request]], [[API Design]], [[Policy and Gate]], [[Laravel Boost]]
---

# Laravel MCP

`laravel/mcp` — MCP(Model Context Protocol, AI 클라이언트가 애플리케이션의 데이터·기능을 "도구"처럼 호출하게 해주는 표준 프로토콜)를 라라벨 방식(attribute, 서비스 컨테이너, 라우트)으로 구현한 공식 패키지.

## 핵심 개념

MCP 서버가 AI 클라이언트(Claude Desktop, Claude Code, Cursor 등)에 노출할 수 있는 것은 세 가지다.

| 구성 요소 | 역할 | 비유 |
|---|---|---|
| Tool | AI가 호출해서 실행하는 함수 (조회/생성/수정) | 컨트롤러의 액션 메서드 |
| Resource | AI가 읽기만 하는 데이터 (URI로 식별) | GET으로 조회하는 정적 리소스 |
| Prompt | 재사용 가능한 대화 템플릿 | 미리 만들어둔 프롬프트 프리셋 |

Server는 이 세 가지를 묶어 하나의 진입점으로 노출하는 컨테이너다.

> `laravel/mcp`는 **내 앱의 업무 기능을 AI 클라이언트에 노출하는 툴킷**이다 — 방향이 반대인 [[Laravel Boost]](AI 에이전트에게 프로젝트 자체의 상태를 알려주는 완제품 MCP 서버)와 헷갈리지 않는다. Boost는 내부적으로 이 MCP 프로토콜 위에서 동작하는 Laravel 공식 제품이다.

## Laravel 구현

### 설치와 서버 등록

```bash
composer require laravel/mcp
php artisan vendor:publish --tag=ai-routes   # routes/ai.php 생성
php artisan make:mcp-server OrderServer
```

```php
// routes/ai.php
use Laravel\Mcp\Facades\Mcp;

Mcp::web('/mcp/orders', OrderServer::class)->middleware(['throttle:mcp']); // 원격 HTTP 클라이언트
Mcp::local('orders', OrderServer::class);                                   // 로컬 CLI 도구 (Claude Code 등)
```

```php
#[Name('Order Server')]
#[Version('1.0.0')]
#[Instructions('주문 조회 및 상태 확인 기능을 제공하는 서버입니다.')]
class OrderServer extends Server
{
    protected array $tools = [LookupOrderTool::class];
    protected array $resources = [];
    protected array $prompts = [];
}
```

### Tool — 가장 많이 쓰는 구성 요소

```php
#[Description('주문번호로 주문 상태, 결제 여부, 배송 예정일을 조회합니다.')]
class LookupOrderTool extends Tool
{
    public function __construct(protected OrderRepository $orders) {} // 컨트롤러처럼 생성자 주입 가능

    public function handle(Request $request): Response
    {
        $validated = $request->validate([
            'order_number' => ['required', 'string', 'max:50'],
        ]);

        $order = Order::where('number', $validated['order_number'])->first();

        return $order
            ? Response::text("주문 {$order->number}: 상태={$order->status}")
            : Response::text("주문번호 {$validated['order_number']}를 찾을 수 없습니다.");
    }

    public function schema(JsonSchema $schema): array
    {
        return [
            'order_number' => $schema->string()->description('조회할 주문번호')->required(),
        ];
    }
}
```

`#[Description]`은 AI가 "이 도구를 언제 써야 하는지" 판단하는 핵심 근거이므로 구체적으로 작성해야 한다. `handle()`에서 실패를 예외가 아니라 `Response::text()`로 안내하면 AI가 사용자에게 그대로 전달할 수 있다.

이름/제목도 기본값은 클래스명에서 자동 추론된다(`LookupOrderTool` → 이름 `lookup-order`, 제목 `Lookup Order Tool`). 커스터마이징하려면 `#[Name]`/`#[Title]`을 붙인다.

```php
use Laravel\Mcp\Server\Attributes\Name;
use Laravel\Mcp\Server\Attributes\Title;

#[Name('lookup-order-by-number')]
#[Title('주문번호로 조회')]
class LookupOrderTool extends Tool { /* ... */ }
```

검증 실패 메시지는 AI가 그대로 읽고 재시도 여부를 판단하므로, 사람보다 AI가 행동할 수 있게 구체적으로 작성한다.

```php
$validated = $request->validate([
    'order_number' => ['required', 'string', 'max:50'],
], [
    'order_number.required' => '조회할 주문번호를 지정해야 합니다. 예: "ORD-20260729-001".',
]);
```

### Tool 출력 스키마와 확장 응답

`schema()`(입력)와 별도로 `outputSchema()`를 정의하면 AI 클라이언트가 응답을 파싱하기 쉬워진다.

```php
public function outputSchema(JsonSchema $schema): array
{
    return [
        'status' => $schema->string()->description('주문 상태')->required(),
        'paid' => $schema->boolean()->description('결제 완료 여부')->required(),
    ];
}
```

단순 텍스트 외에 여러 응답 형태를 지원한다.

```php
// 구조화된 데이터 — AI가 파싱 가능한 JSON과 함께 반환
return Response::structured(['status' => 'shipped', 'paid' => true]);

// 커스텀 텍스트 + 구조화 데이터를 함께
return Response::make(Response::text('주문이 배송 중입니다.'))
    ->withStructuredContent(['status' => 'shipped', 'paid' => true]);

// 여러 개의 응답을 배열로 반환
return [Response::text('요약: 배송 중'), Response::text('상세: ...')];
```

오래 걸리는 작업(대량 조회, 배치 처리)은 `handle()`에서 `Generator`를 반환해 중간 진행 상황을 스트리밍할 수 있다 — Web 서버에서는 자동으로 SSE(Server-Sent Events) 스트림이 열린다.

```php
use Generator;

/** @return \Generator<int, \Laravel\Mcp\Response> */
public function handle(Request $request): Generator
{
    $orderNumbers = $request->array('order_numbers');

    foreach ($orderNumbers as $index => $number) {
        yield Response::notification('processing/progress', [
            'current' => $index + 1,
            'total' => count($orderNumbers),
        ]);

        yield Response::text($this->lookup($number));
    }
}
```

### 부작용 어노테이션과 조건부 등록

```php
#[IsReadOnly(true)]
#[IsIdempotent(true)]
class LookupOrderTool extends Tool {}

#[IsReadOnly(false)]
#[IsDestructive(true)] // 주문 취소, 회원 삭제 등
class CancelOrderTool extends Tool
{
    // true를 반환해야 이 도구가 AI 클라이언트 목록에 노출됨
    public function shouldRegister(Request $request): bool
    {
        return $request?->user()?->can('cancel-orders') ?? false;
    }
}
```

`IsReadOnly`/`IsDestructive`/`IsIdempotent`는 강제 규칙이 아니라 AI 클라이언트에 주는 힌트지만, 위험한 도구를 가볍게 반복 호출하는 것을 막는 데 도움이 된다. `shouldRegister()`는 [[Policy and Gate|Gate/Policy]]와 같은 인가 판단을 도구 노출 여부에 그대로 적용하는 지점이다.

### Resource — 정적 URI와 Template

```php
#[Uri('order://policy/refund')]
#[MimeType('text/plain')]
#[Description('환불/취소 정책 전문')]
class OrderPolicyResource extends Resource
{
    public function handle(): Response
    {
        return Response::text('주문 후 7일 이내, 미배송 건에 한해 전액 환불이 가능합니다...');
    }
}
```

URI에 변수가 필요하면(예: 사용자별 프로필) `HasUriTemplate`를 구현하고 `UriTemplate`로 패턴을 선언한다 — 정적 리소스가 아니라 **템플릿**으로 등록되어, 패턴에 매칭되는 모든 URI 요청을 이 클래스 하나가 처리한다.

```php
use Laravel\Mcp\Server\Contracts\HasUriTemplate;
use Laravel\Mcp\Support\UriTemplate;

class UserOrdersResource extends Resource implements HasUriTemplate
{
    public function uriTemplate(): UriTemplate
    {
        return new UriTemplate('order://users/{userId}/orders');
    }

    public function handle(Request $request): Response
    {
        $userId = $request->get('userId'); // 템플릿 변수가 자동으로 request에 병합됨

        return Response::text("사용자 {$userId}의 주문 목록...");
    }
}
```

Resource는 Tool/Prompt와 달리 **입력 스키마를 정의할 수 없다** — `handle()`에서 URI 템플릿 변수만 접근 가능하다. 응답은 `text`/`blob`(바이너리, MIME은 `#[MimeType]`을 따름)/`error` 세 종류.

`#[Audience]`(`Role::User`/`Role::Assistant`)/`#[Priority]`(0.0~1.0)/`#[LastModified]` 어노테이션으로 AI 클라이언트에 리소스 메타정보를 추가로 전달할 수 있다.

### Prompt — 인자를 받는 대화 템플릿

```php
use Laravel\Mcp\Server\Prompts\Argument;

class SummarizeOrderPrompt extends Prompt
{
    /** @return array<int, \Laravel\Mcp\Server\Prompts\Argument> */
    public function arguments(): array
    {
        return [
            new Argument(name: 'tone', description: '요약 어조 (정중한/캐주얼 등)', required: true),
        ];
    }

    public function handle(Request $request): array
    {
        $tone = $request->string('tone', '정중한');

        return [
            Response::text("당신은 {$tone} 어조로 주문 상태를 요약해주는 고객지원 어시스턴트입니다.")->asAssistant(),
            Response::text('최근 주문 상태를 요약해줘.'),
        ];
    }
}
```

`asAssistant()`를 붙인 응답은 AI 쪽 발화(시스템 프롬프트 역할)로, 일반 `Response::text()`는 사용자 발화로 취급된다. Prompt는 Tool/Resource보다 사용 빈도가 낮다.

### 메타데이터(`_meta`)

일부 MCP 클라이언트가 요구하는 [MCP 스펙의 `_meta` 필드](https://modelcontextprotocol.io/specification/2025-06-18/basic#meta)를 세 레벨로 붙일 수 있다.

```php
// 1) 개별 응답 콘텐츠
return Response::text('배송 중입니다.')->withMeta(['source' => 'order-api', 'cached' => true]);

// 2) 응답 envelope 전체
return Response::make(Response::text('배송 중입니다.'))->withMeta(['request_id' => '12345']);

// 3) Tool/Resource/Prompt 클래스 자체
class LookupOrderTool extends Tool
{
    protected ?array $meta = ['version' => '2.0'];
}
```

### 인증과 인가

Web 서버는 일반 라우트처럼 미들웨어로 보호한다.

```php
use Laravel\Mcp\Facades\Mcp;

// Sanctum 토큰 — 클라이언트가 Authorization: Bearer <token> 헤더를 보내야 함
Mcp::web('/mcp/orders', OrderServer::class)->middleware('auth:sanctum');

// OAuth 2.1 (Passport) — MCP 스펙이 공식 문서화한 방식, 대부분의 MCP 클라이언트가 지원
Mcp::oauthRoutes();
Mcp::web('/mcp/orders', OrderServer::class)->middleware('auth:api');
```

Passport를 새로 설치한다면 `php artisan vendor:publish --tag=mcp-views`로 인가 화면을 퍼블리시하고 `AppServiceProvider::boot()`에서 `Passport::authorizationView(...)`로 연결한다. 이미 Sanctum을 쓰는 앱이라면 Passport 전용 MCP 클라이언트가 필요해지기 전까지는 Sanctum만으로 충분하다.

인가는 `$request->user()`로 [[Policy and Gate|Gate/Policy]]를 그대로 호출하면 된다 — `shouldRegister()`로 도구 자체를 숨기거나, `handle()` 안에서 명시적으로 거부한다.

```php
public function handle(Request $request): Response
{
    if (! $request->user()->can('cancel-orders')) {
        return Response::error('권한이 없습니다.');
    }
    // ...
}
```

### 테스트 — MCP Inspector와 유닛 테스트

```bash
php artisan mcp:inspector mcp/orders   # Web 서버
php artisan mcp:inspector orders       # Local 서버
```

브라우저 기반 테스트 UI와 실제 AI 클라이언트에 붙여넣을 접속 설정값을 함께 보여준다. Pest 유닛 테스트도 가능하다.

```php
test('주문 조회 도구', function () {
    $response = OrderServer::actingAs($user)->tool(LookupOrderTool::class, ['order_number' => 'ORD-1']);

    $response
        ->assertOk()
        ->assertSee('배송 중')
        ->assertHasNoErrors();
});
```

`actingAs($user)`로 인증된 사용자를 시뮬레이션하고, `OrderServer::prompt(...)`/`OrderServer::resource(...)`로 Prompt/Resource도 같은 방식으로 테스트한다. 그 외 `assertHasErrors()`, `assertName()`/`assertTitle()`/`assertDescription()`, 스트리밍 응답 확인용 `assertSentNotification()`/`assertNotificationCount()`, 디버깅용 `dd()`/`dump()` 어설션이 있다.

## DDD 관점에서의 활용

- Tool의 `handle()` 구조(입력 검증 → 처리 → 응답)는 [[Form Request]] + 컨트롤러 조합과 본질적으로 같은 역할이다 — 다만 `Laravel\Mcp\Request::validate()`는 HTTP `FormRequest` 클래스를 그대로 주입받는 방식이 아니므로, 규칙 자체를 공유하고 싶다면 `rules()` 배열을 별도 클래스나 상수로 추출해 API 컨트롤러와 MCP Tool 양쪽에서 참조하는 편이 안전하다. 처리 로직은 Application Service를 공유하면 된다.
- Tool을 [[API Design]]의 "또 다른 클라이언트"로 보면 이해하기 쉽다 — REST API가 HTTP 클라이언트에게 노출하는 유스케이스를, MCP Tool은 AI 클라이언트에게 같은 방식으로 노출한다. 두 계층 모두 도메인 로직을 직접 담지 않고 Application Service/Action을 호출하는 얇은 어댑터로 유지하는 원칙이 동일하게 적용된다.
- `#[Bind]`/`#[BindWhen]`([[Container Binding Attributes]])으로 등록한 인터페이스는 Tool 생성자에서도 동일하게 주입된다 — MCP 계층이 별도의 의존성 배선을 요구하지 않는다.

## 개발 체크리스트

새 Tool/Resource/Prompt를 만들 때 순서대로 확인한다.

1. `make:mcp-*`로 생성했는가?
2. **서버의 `$tools`/`$resources`/`$prompts` 배열에 등록했는가?** — 가장 흔히 빠뜨리는 단계. 클래스만 만들고 등록을 빼먹으면 AI 클라이언트에 전혀 보이지 않는다.
3. `#[Description]`을 구체적으로 작성했는가? (자동 생성되지 않으며, AI의 호출 판단 근거다)
4. Tool이라면: 입력 스키마(`schema()`)와 필요한 검증(`$request->validate()`)을 정의했는가?
5. Tool이 데이터를 변경/삭제한다면: `#[IsReadOnly(false)]` + `#[IsDestructive(true)]`를 붙였는가?
6. 인증이 필요한 서버인가? `routes/ai.php`에 `auth:sanctum`/`auth:api` 미들웨어가 붙어 있는가?
7. 사용자별 접근 제한이 필요한가? `handle()` 안에서 `$request->user()->can(...)`을 체크했는가, 또는 `shouldRegister()`로 노출 자체를 막아야 하는가?
8. `php artisan mcp:inspector <name>`으로 실제 동작을 확인했는가?

## 주의사항 / 안티패턴

- **Tool `handle()`에 도메인 로직을 직접 작성하지 말 것** — 컨트롤러와 마찬가지로 얇게 유지하고 Application Service/Action에 위임한다. 그래야 REST API와 MCP Tool이 같은 유스케이스 코드를 재사용할 수 있다.
- **파괴적 동작에 `#[IsDestructive]`/`shouldRegister()` 인가 체크를 빠뜨리지 말 것** — AI 클라이언트는 사람보다 훨씬 빠르고 반복적으로 도구를 호출할 수 있다.
- **`Mcp::web()`에 인증/스로틀 미들웨어 없이 배포하지 말 것** — 원격 클라이언트가 붙는 엔드포인트이므로 일반 API 라우트와 동일한 보안 기준을 적용한다.

## Laravel Boost와 함께 쓰기 (실전 팁)

`laravel/mcp`와 `laravel/boost`는 별도 프로젝트가 아니라 **같은 프로젝트에 병행 설치**하는 것이 일반적이다 — "Boost로 MCP 서버를 만든다"가 아니라 "MCP 서버(`laravel/mcp`)를 만드는 동안 Boost가 AI 코딩을 돕는다"는 관계다.

```bash
composer require laravel/mcp          # MCP 서버를 실제로 만드는 패키지 — 운영에도 배포됨
composer require laravel/boost --dev  # 이 프로젝트를 짜는 동안 AI에게 컨텍스트를 주는 개발 도구
php artisan boost:install
```

[[Laravel Boost]]의 Agent Skills는 `composer.json`에 설치된 패키지를 감지해 자동으로 스킬을 설치하는데, 그 지원 목록에 `MCP`가 포함되어 있다 — 즉 같은 프로젝트에 `laravel/mcp`가 설치되어 있으면 Boost가 이를 감지해 "Tool/Resource/Prompt를 어떻게 올바르게 작성하는지"에 대한 상세 가이드를 AI 에이전트에게 자동으로 제공한다. Boost는 선택 사항이며, `laravel/mcp`만으로도 MCP 서버를 완전히 만들 수 있다.

## 참고

- [[Service Provider]] / [[Container]] — Tool/Resource/Prompt에 그대로 적용되는 의존성 주입 메커니즘
- [[Form Request]] — Tool의 입력 검증과 같은 층위의 패턴
- [[API Design]] — REST API와 MCP Tool을 "같은 유스케이스의 다른 클라이언트"로 보는 관점
- [[Container Binding Attributes]] — Tool 생성자에 주입되는 인터페이스 바인딩을 attribute로 선언하는 방법
- [[Policy and Gate]] — `shouldRegister()`/`handle()` 내 인가 체크에 그대로 재사용하는 패턴
- [[Laravel Boost]] — 같은 MCP 프로토콜 위에서 반대 방향(프로젝트 컨텍스트를 AI에 제공)으로 동작하는 Laravel 공식 제품. 같은 프로젝트에 병행 설치하는 실전 팁은 위 "Laravel Boost와 함께 쓰기" 참고
- 소스: Laravel MCP 정리 (사내 요약), Laravel MCP Official Docs Reference (공식 블로그 + 12.x/13.x 문서 종합)

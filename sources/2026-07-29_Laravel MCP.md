# Laravel MCP 정리

> MCP(Model Context Protocol)는 Claude, Cursor, GitHub Copilot 같은 AI 클라이언트가
> 애플리케이션의 실제 데이터·기능을 "도구"처럼 호출하게 해주는 표준 프로토콜.
> Laravel MCP는 이 프로토콜을 라라벨 방식(어트리뷰트, 서비스 컨테이너, 라우트)으로 구현한 공식 패키지.
> 공식 문서: https://laravel.com/docs/13.x/mcp

---

## 0. 언제 쓰나

- 사내 라라벨 앱(주문/재고/CRM 등)의 데이터를 Claude Desktop, Claude Code, Cursor 같은 AI 클라이언트가
  직접 조회하게 하고 싶을 때
- AI 에이전트가 "추측"이 아니라 실제 DB 값을 근거로 답하게 만들고 싶을 때
- 라라벨 패키지를 만들어서 다른 개발자의 AI 도구(Claude Code 등)에 기능을 노출하고 싶을 때

MCP 서버가 노출할 수 있는 것은 세 가지입니다.

| 구성 요소 | 역할 | 비유 |
|---|---|---|
| Tool | AI가 호출해서 실행하는 함수 (조회, 생성, 수정 등) | 컨트롤러의 액션 메서드 |
| Resource | AI가 읽기만 하는 데이터 (URI로 식별) | GET으로 조회하는 정적 리소스 |
| Prompt | 재사용 가능한 대화 템플릿 | 미리 만들어둔 프롬프트 프리셋 |

---

## 1. 설치

```bash
composer require laravel/mcp

# routes/ai.php 파일을 생성 (MCP 서버를 등록하는 곳)
php artisan vendor:publish --tag=ai-routes
```

---

## 2. 서버(Server) 만들기

서버는 여러 개의 Tool/Resource/Prompt를 묶어서 AI 클라이언트에게 노출하는 진입점입니다.

```bash
php artisan make:mcp-server OrderServer
```

```php
// app/Mcp/Servers/OrderServer.php

namespace App\Mcp\Servers;

use App\Mcp\Tools\LookupOrderTool;
use Laravel\Mcp\Server\Attributes\Instructions;
use Laravel\Mcp\Server\Attributes\Name;
use Laravel\Mcp\Server\Attributes\Version;
use Laravel\Mcp\Server;

#[Name('Order Server')]                 // AI 클라이언트에 표시되는 서버 이름
#[Version('1.0.0')]
#[Instructions('주문 조회 및 상태 확인 기능을 제공하는 서버입니다.')] // AI에게 전달되는 서버 사용법 힌트
class OrderServer extends Server
{
    /**
     * 이 서버가 제공하는 도구(Tool) 목록
     * @var array<int, class-string<\Laravel\Mcp\Server\Tool>>
     */
    protected array $tools = [
        LookupOrderTool::class,
    ];

    /**
     * 이 서버가 제공하는 리소스(Resource) 목록
     * @var array<int, class-string<\Laravel\Mcp\Server\Resource>>
     */
    protected array $resources = [
        // OrderPolicyResource::class,
    ];

    /**
     * 이 서버가 제공하는 프롬프트(Prompt) 목록
     * @var array<int, class-string<\Laravel\Mcp\Server\Prompt>>
     */
    protected array $prompts = [
        // SummarizeOrderPrompt::class,
    ];
}
```

---

## 3. 도구(Tool) 만들기 — 가장 많이 쓰는 부분

```bash
php artisan make:mcp-tool LookupOrderTool
```

### 3-1. 기본 구조: 스키마 정의 + 처리 로직

```php
// app/Mcp/Tools/LookupOrderTool.php

namespace App\Mcp\Tools;

use App\Models\Order;
use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Mcp\Request;
use Laravel\Mcp\Response;
use Laravel\Mcp\Server\Attributes\Description;
use Laravel\Mcp\Server\Tool;

// Description은 AI가 "이 도구를 언제 써야 하는지" 판단하는 핵심 근거이므로 꼭 구체적으로 작성
#[Description('주문번호로 주문 상태, 결제 여부, 배송 예정일을 조회합니다.')]
class LookupOrderTool extends Tool
{
    /**
     * AI 클라이언트가 이 도구를 호출할 때 실제로 실행되는 메서드.
     */
    public function handle(Request $request): Response
    {
        // 1) 라라벨 Validator로 입력값 검증 (AI가 잘못된 값을 넣었을 때 에러 메시지로 재시도 유도)
        $validated = $request->validate([
            'order_number' => ['required', 'string', 'max:50'],
        ]);

        $order = Order::where('number', $validated['order_number'])->first();

        if (! $order) {
            // 텍스트 응답으로 실패 사유를 알려주면, AI가 사용자에게 그대로 안내할 수 있음
            return Response::text("주문번호 {$validated['order_number']}를 찾을 수 없습니다.");
        }

        return Response::text(
            "주문 {$order->number}: 상태={$order->status}, 결제완료={$order->paid_at ? '예' : '아니오'}, 배송예정일={$order->eta}"
        );
    }

    /**
     * 도구가 받는 입력값의 스키마. AI 클라이언트는 이 스키마를 보고 호출 형식을 결정함.
     * @return array<string, \Illuminate\JsonSchema\Types\Type>
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'order_number' => $schema->string()
                ->description('조회할 주문번호 (예: ORD-20260729-001)')
                ->required(),
        ];
    }
}
```

서버에 등록하는 것도 잊지 말 것.

```php
// app/Mcp/Servers/OrderServer.php
protected array $tools = [
    LookupOrderTool::class,
];
```

### 3-2. 의존성 주입 — 컨트롤러처럼 서비스 컨테이너가 그대로 동작

```php
namespace App\Mcp\Tools;

use App\Repositories\OrderRepository;
use Laravel\Mcp\Request;
use Laravel\Mcp\Response;
use Laravel\Mcp\Server\Tool;

class LookupOrderTool extends Tool
{
    // 생성자 주입도 가능하고,
    public function __construct(
        protected OrderRepository $orders,
    ) {}

    // handle() 메서드 인자로도 주입받을 수 있음 (둘 다 지원)
    public function handle(Request $request, OrderRepository $orders): Response
    {
        // $this->orders 또는 $orders 둘 다 사용 가능
        return Response::text('...');
    }
}
```

### 3-3. 도구 어노테이션 — AI에게 "부작용이 있는지" 알려주기

```php
use Laravel\Mcp\Server\Tools\Annotations\IsDestructive;
use Laravel\Mcp\Server\Tools\Annotations\IsIdempotent;
use Laravel\Mcp\Server\Tools\Annotations\IsReadOnly;
use Laravel\Mcp\Server\Tool;

#[IsReadOnly(true)]     // 조회만 하고 데이터를 변경하지 않음 (LookupOrderTool 같은 경우)
#[IsIdempotent(true)]   // 같은 인자로 여러 번 호출해도 결과/부작용이 동일함
class LookupOrderTool extends Tool
{
    //
}
```

```php
// 반대로 데이터를 변경/삭제하는 도구라면 이렇게 표시
#[IsReadOnly(false)]
#[IsDestructive(true)]  // 예: 주문 취소, 회원 삭제 같은 도구
class CancelOrderTool extends Tool
{
    //
}
```

이 값들은 강제 규칙이 아니라 AI 클라이언트에게 주는 "힌트"지만, 실수로 위험한 도구를 가볍게 반복 호출하는 걸 막는 데 도움이 됩니다.

### 3-4. 조건부 등록 — 특정 조건에서만 도구 노출

```php
use Laravel\Mcp\Request;
use Laravel\Mcp\Server\Tool;

class CancelOrderTool extends Tool
{
    /**
     * true를 반환해야 이 도구가 AI 클라이언트 목록에 노출됨.
     * 예: 관리자 권한이 있는 사용자에게만 "주문 취소" 도구를 노출.
     */
    public function shouldRegister(Request $request): bool
    {
        return $request?->user()?->can('cancel-orders') ?? false;
    }
}
```

---

## 4. 리소스(Resource) 만들기 — 읽기 전용 데이터 노출

정적인 문서나 데이터를 AI가 "컨텍스트"로 참고하게 하고 싶을 때 씁니다. (Tool처럼 실행하는 게 아니라 그냥 읽는 용도)

```bash
php artisan make:mcp-resource OrderPolicyResource
```

```php
namespace App\Mcp\Resources;

use Laravel\Mcp\Server\Attributes\Description;
use Laravel\Mcp\Server\Attributes\MimeType;
use Laravel\Mcp\Server\Attributes\Uri;
use Laravel\Mcp\Server\Resource;

#[Uri('order://policy/refund')]         // 리소스를 식별하는 고유 URI (생략 시 이름 기반으로 자동 생성)
#[MimeType('text/plain')]               // 기본값은 text/plain
#[Description('환불/취소 정책 전문')]
class OrderPolicyResource extends Resource
{
    public function handle(): \Laravel\Mcp\Response
    {
        return \Laravel\Mcp\Response::text(
            '주문 후 7일 이내, 미배송 건에 한해 전액 환불이 가능합니다...'
        );
    }
}
```

URI 안에 변수를 넣어 여러 리소스를 하나의 클래스로 처리하고 싶다면 `HasUriTemplate`를 구현합니다.

```php
namespace App\Mcp\Resources;

use Laravel\Mcp\Request;
use Laravel\Mcp\Response;
use Laravel\Mcp\Server\Contracts\HasUriTemplate;
use Laravel\Mcp\Server\Resource;
use Laravel\Mcp\Support\UriTemplate;

class UserProfileResource extends Resource implements HasUriTemplate
{
    // order://users/{userId}/profile 같은 URI 패턴을 하나의 클래스로 처리
    public function uriTemplate(): UriTemplate
    {
        return new UriTemplate('order://users/{userId}/profile');
    }

    public function handle(Request $request): Response
    {
        $userId = $request->get('userId'); // URI에서 추출된 변수

        return Response::text("사용자 {$userId}의 프로필...");
    }
}
```

---

## 5. 프롬프트(Prompt) — 참고만 하는 정도로 간단히

미리 정의된 대화 흐름(시스템 메시지 + 사용자 메시지)을 AI 클라이언트에 제공하고 싶을 때 씁니다. Tool/Resource에 비해 사용 빈도는 낮습니다.

```php
namespace App\Mcp\Prompts;

use Laravel\Mcp\Request;
use Laravel\Mcp\Response;
use Laravel\Mcp\Server\Prompt;

class SummarizeOrderPrompt extends Prompt
{
    public function handle(Request $request): array
    {
        $tone = $request->string('tone', '정중한'); // 기본값 지정 가능

        return [
            // asAssistant(): AI 쪽 발화로 취급 (시스템 프롬프트 역할)
            Response::text("당신은 {$tone} 어조로 주문 상태를 요약해주는 고객지원 어시스턴트입니다.")->asAssistant(),
            // 일반 Response::text()는 사용자 발화로 취급됨
            Response::text('최근 주문 상태를 요약해줘.'),
        ];
    }
}
```

---

## 6. 서버 등록 (`routes/ai.php`) — Web vs Local

```php
// routes/ai.php

use App\Mcp\Servers\OrderServer;
use Laravel\Mcp\Facades\Mcp;

// (1) Web 서버: 원격 AI 클라이언트가 HTTP POST로 접속 (일반적인 경우)
Mcp::web('/mcp/orders', OrderServer::class)
    ->middleware(['throttle:mcp']); // 일반 라우트처럼 미들웨어 적용 가능

// (2) Local 서버: Claude Code 등 로컬 CLI 도구가 커맨드로 실행 (mcp:start를 직접 돌릴 필요 없음)
Mcp::local('orders', OrderServer::class);
```

- `Mcp::web(...)` : Claude Desktop의 "커넥터" 등록처럼 원격에서 HTTP로 붙는 경우
- `Mcp::local(...)` : Claude Code처럼 같은 머신에서 실행되는 AI 도구와 연동하는 경우 (Laravel Boost가 이 방식)

---

## 7. 테스트 — MCP Inspector

```bash
# Web 서버
php artisan mcp:inspector mcp/orders

# Local 서버 (등록한 이름 그대로)
php artisan mcp:inspector orders
```

Inspector를 실행하면 브라우저 기반 테스트 UI와, 실제 AI 클라이언트(Claude 등)에 그대로 붙여넣을 수 있는 접속 설정값을 함께 보여줍니다. 인증 미들웨어를 걸어둔 경우 Authorization 헤더 등을 Inspector에 직접 입력해서 확인합니다.

간단한 유닛 테스트도 가능합니다 (Pest 기준).

```php
test('주문 조회 도구', function () {
    $response = OrderServer::tool(LookupOrderTool::class, [
        'order_number' => 'ORD-20260729-001',
    ]);

    $response->assertOk();
});
```

---

## 8. 전체 흐름 요약

```
1. composer require laravel/mcp
2. php artisan vendor:publish --tag=ai-routes   →  routes/ai.php 생성
3. php artisan make:mcp-server OrderServer      →  app/Mcp/Servers/OrderServer.php
4. php artisan make:mcp-tool LookupOrderTool    →  app/Mcp/Tools/LookupOrderTool.php
5. OrderServer의 $tools 배열에 LookupOrderTool 등록
6. routes/ai.php 에서 Mcp::web() 또는 Mcp::local() 로 서버 등록
7. php artisan mcp:inspector 로 동작 확인
8. Claude Desktop / Claude Code 등에 접속 정보 등록 → AI가 실제 라라벨 앱 데이터를 조회
```

---

## 참고

- [Laravel MCP | Laravel 13.x 공식 문서](https://laravel.com/docs/13.x/mcp)
- [Laravel MCP Beta is Released - Laravel News](https://laravel-news.com/laravel-mcp-beta)
- [Model Context Protocol 공식 사이트](https://modelcontextprotocol.io/docs/getting-started/intro)

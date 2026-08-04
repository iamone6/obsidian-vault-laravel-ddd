# Laravel MCP 개발 레퍼런스 (공식 문서 종합)

> **출처**
> - [Introducing Laravel MCP: Build with the Universal AI Standard](https://laravel.com/blog/introducing-laravel-mcp-build-with-the-universal-ai-standard) (Laravel 공식 블로그, Ashley Hindle, 2025-09-18)
> - [Laravel MCP 공식 문서](https://laravel.com/docs/12.x/mcp) (12.x 기준으로 페치됨. 문서 상단에 "13.x로 업그레이드하라"는 권장 배너만 있고, API 변경 안내는 없음 — 최신 버전은 [13.x 문서](https://laravel.com/docs/13.x/mcp) 참고)
> - 이 문서는 **Claude Code가 실제로 Laravel MCP 서버/Tool/Resource/Prompt를 만들 때 바로 참고할 수 있도록** 코드 예제 위주로 정리했습니다. 공식 문서의 설명을 그대로 옮기기보다, "이 상황에서 뭘 써야 하는가"를 빠르게 찾을 수 있게 재구성했습니다.

---

## 0. 30초 요약 — 이것만 알면 시작할 수 있다

```bash
composer require laravel/mcp
php artisan vendor:publish --tag=ai-routes   # routes/ai.php 생성
php artisan make:mcp-server OrderServer      # app/Mcp/Servers/OrderServer.php
php artisan make:mcp-tool LookupOrderTool    # app/Mcp/Tools/LookupOrderTool.php
```

MCP 서버가 노출하는 세 가지:

| 구성 요소 | 역할 | 생성 명령 | 기본 위치 |
|---|---|---|---|
| **Tool** | AI가 호출해서 실행하는 함수 (조회/생성/수정) | `make:mcp-tool` | `app/Mcp/Tools/` |
| **Resource** | AI가 읽기만 하는 데이터 (URI로 식별) | `make:mcp-resource` | `app/Mcp/Resources/` |
| **Prompt** | 재사용 가능한 대화 템플릿 | `make:mcp-prompt` | `app/Mcp/Prompts/` |

Server는 이 셋을 묶어 AI 클라이언트에 노출하는 진입점이다 (`app/Mcp/Servers/`).

```php
// routes/ai.php
use App\Mcp\Servers\OrderServer;
use Laravel\Mcp\Facades\Mcp;

Mcp::web('/mcp/orders', OrderServer::class)->middleware(['throttle:mcp']); // 원격 HTTP 클라이언트
Mcp::local('orders', OrderServer::class);                                   // 로컬 CLI 도구 (Claude Code 등)
```

지원 환경: Laravel 10/11/12(+13), **PHP 8.1+**. Laravel 자체 팀이 만든 [Laravel Boost](https://blog.laravel.com/announcing-laravel-boost)가 이 패키지로 구현되어 실전 검증됨. OAuth([Passport](https://laravel.com/docs/12.x/passport))와 토큰 인증([Sanctum](https://laravel.com/docs/12.x/sanctum))을 기본 지원.

---

## 1. Laravel MCP란

[MCP(Model Context Protocol)](https://modelcontextprotocol.io/docs/getting-started/intro)는 ChatGPT, Claude, Cursor 같은 AI 클라이언트가 애플리케이션의 데이터·기능을 표준화된 방식으로 호출하게 해주는 프로토콜이다. `laravel/mcp`는 이 프로토콜을 라라벨다운 문법(attribute, 서비스 컨테이너, 라우트)으로 구현한 공식 패키지다.

Laravel 팀은 "AI 채팅이 하루 30억 개 이상의 메시지를 처리하는 지금, MCP는 웹·API와 나란히 앱 기능의 핵심 진입점이 될 것"이라는 관점에서 이 패키지를 발표했다. 참고용 데모 앱 [Locket](https://github.com/laravel/locket)([라이브](https://locket.laravel.cloud))은 웹 인터페이스·JSON API·MCP 서버 세 가지를 모두 갖춘 실제 제품 예시다.

---

## 2. 설치

```bash
composer require laravel/mcp
php artisan vendor:publish --tag=ai-routes
```

`vendor:publish` 명령이 `routes/ai.php`를 생성한다 — 이후 모든 MCP 서버를 이 파일에서 등록한다.

---

## 3. 서버(Server) 만들기

```bash
php artisan make:mcp-server WeatherServer
```

`app/Mcp/Servers/WeatherServer.php`가 생성되며, `Laravel\Mcp\Server`를 상속한다.

```php
<?php

namespace App\Mcp\Servers;

use Laravel\Mcp\Server\Attributes\Instructions;
use Laravel\Mcp\Server\Attributes\Name;
use Laravel\Mcp\Server\Attributes\Version;
use Laravel\Mcp\Server;

#[Name('Weather Server')]                                          // AI 클라이언트에 표시되는 이름
#[Version('1.0.0')]
#[Instructions('This server provides weather information and forecasts.')] // AI에게 전달되는 서버 사용법 힌트
class WeatherServer extends Server
{
    /** @var array<int, class-string<\Laravel\Mcp\Server\Tool>> */
    protected array $tools = [
        // GetCurrentWeatherTool::class,
    ];

    /** @var array<int, class-string<\Laravel\Mcp\Server\Resource>> */
    protected array $resources = [
        // WeatherGuidelinesResource::class,
    ];

    /** @var array<int, class-string<\Laravel\Mcp\Server\Prompt>> */
    protected array $prompts = [
        // DescribeWeatherPrompt::class,
    ];
}
```

Tool/Resource/Prompt 클래스를 만든 뒤 반드시 해당 `$tools`/`$resources`/`$prompts` 배열에 등록해야 AI 클라이언트에 노출된다 — **클래스만 만들고 등록을 빼먹는 실수가 가장 흔하다.**

### 3.1 서버 등록 — Web vs Local

`routes/ai.php`에서 두 가지 방식으로 등록한다.

**Web 서버** — 원격 AI 클라이언트가 HTTP POST로 접속. 가장 흔한 경우.

```php
use App\Mcp\Servers\WeatherServer;
use Laravel\Mcp\Facades\Mcp;

Mcp::web('/mcp/weather', WeatherServer::class);

// 일반 라우트처럼 미들웨어 적용 가능
Mcp::web('/mcp/weather', WeatherServer::class)
    ->middleware(['throttle:mcp']);
```

**Local 서버** — Claude Code 같은 로컬 CLI 도구용. Artisan 커맨드로 실행됨.

```php
Mcp::local('weather', WeatherServer::class);
```

등록 후에는 `mcp:start`를 직접 실행할 필요가 없다 — MCP 클라이언트(AI 에이전트)가 알아서 서버를 구동하거나, 테스트용으로 [MCP Inspector](#10-테스트)를 쓴다.

---

## 4. Tool — AI가 호출해서 "행동"하게 하는 구성 요소

```bash
php artisan make:mcp-tool CurrentWeatherTool
```

### 4.1 기본 구조

```php
<?php

namespace App\Mcp\Tools;

use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Mcp\Request;
use Laravel\Mcp\Response;
use Laravel\Mcp\Server\Attributes\Description;
use Laravel\Mcp\Server\Tool;

#[Description('Fetches the current weather forecast for a specified location.')]
class CurrentWeatherTool extends Tool
{
    public function handle(Request $request): Response
    {
        $location = $request->get('location');

        // Get weather...

        return Response::text('The weather is...');
    }

    /** @return array<string, \Illuminate\JsonSchema\Types\Type> */
    public function schema(JsonSchema $schema): array
    {
        return [
            'location' => $schema->string()
                ->description('The location to get the weather for.')
                ->required(),
        ];
    }
}
```

생성 후 서버의 `$tools` 배열에 등록:

```php
protected array $tools = [
    CurrentWeatherTool::class,
];
```

### 4.2 이름 / 제목 / 설명 커스터마이징

기본값은 클래스명에서 추론된다 — `CurrentWeatherTool` → 이름 `current-weather`, 제목 `Current Weather Tool`.

```php
use Laravel\Mcp\Server\Attributes\Name;
use Laravel\Mcp\Server\Attributes\Title;

#[Name('get-optimistic-weather')]
#[Title('Get Optimistic Weather Forecast')]
class CurrentWeatherTool extends Tool { /* ... */ }
```

`#[Description]`은 **자동 생성되지 않으므로 반드시 직접 작성**해야 한다 — AI가 "이 도구를 언제 써야 하는지" 판단하는 핵심 근거다. 모호하게 쓰면 AI가 엉뚱한 상황에서 호출하거나 아예 호출하지 않는다.

### 4.3 입력 스키마 — `Illuminate\Contracts\JsonSchema\JsonSchema`

```php
public function schema(JsonSchema $schema): array
{
    return [
        'location' => $schema->string()
            ->description('The location to get the weather for.')
            ->required(),

        'units' => $schema->string()
            ->enum(['celsius', 'fahrenheit'])
            ->description('The temperature units to use.')
            ->default('celsius'),
    ];
}
```

### 4.4 출력 스키마 — AI 클라이언트가 결과를 파싱하기 쉽게

```php
public function outputSchema(JsonSchema $schema): array
{
    return [
        'temperature' => $schema->number()->description('Temperature in Celsius')->required(),
        'conditions'  => $schema->string()->description('Weather conditions')->required(),
        'humidity'    => $schema->integer()->description('Humidity percentage')->required(),
    ];
}
```

### 4.5 검증 — 라라벨 Validator 그대로 사용

JSON Schema는 기본 구조만 강제하므로, 복잡한 규칙은 `handle()` 안에서 일반 라라벨 검증으로 처리한다.

```php
public function handle(Request $request): Response
{
    $validated = $request->validate([
        'location' => 'required|string|max:100',
        'units' => 'in:celsius,fahrenheit',
    ]);

    // ...
}
```

**검증 실패 시 에러 메시지를 AI가 그대로 보고 재시도하므로, 메시지를 사람이 아니라 AI가 이해하고 행동할 수 있게 구체적으로 작성해야 한다:**

```php
$validated = $request->validate([
    'location' => ['required', 'string', 'max:100'],
    'units' => 'in:celsius,fahrenheit',
], [
    'location.required' => 'You must specify a location to get the weather for. For example, "New York City" or "Tokyo".',
    'units.in' => 'You must specify either "celsius" or "fahrenheit" for the units.',
]);
```

### 4.6 의존성 주입 — 컨트롤러와 완전히 동일

생성자 주입, `handle()` 메서드 인자 주입 둘 다 지원된다.

```php
class CurrentWeatherTool extends Tool
{
    public function __construct(
        protected WeatherRepository $weather, // 생성자 주입
    ) {}

    public function handle(Request $request, WeatherRepository $weather): Response
    {
        // $this->weather 또는 $weather 둘 다 사용 가능
        $forecast = $weather->getForecastFor($request->get('location'));
        // ...
    }
}
```

### 4.7 어노테이션 — AI에게 "부작용이 있는지" 알려주는 힌트

```php
use Laravel\Mcp\Server\Tools\Annotations\IsDestructive;
use Laravel\Mcp\Server\Tools\Annotations\IsIdempotent;
use Laravel\Mcp\Server\Tools\Annotations\IsOpenWorld;
use Laravel\Mcp\Server\Tools\Annotations\IsReadOnly;

#[IsReadOnly(true)]     // 환경을 변경하지 않음
#[IsDestructive(false)] // 파괴적 갱신 아님 (IsReadOnly가 false일 때만 의미 있음)
#[IsOpenWorld(false)]   // 외부 시스템과 상호작용하지 않음
#[IsIdempotent(true)]   // 같은 인자로 여러 번 호출해도 결과가 같음
class CurrentWeatherTool extends Tool { /* ... */ }
```

| 어노테이션 | 타입 | 의미 |
|---|---|---|
| `#[IsReadOnly]` | bool | 환경을 변경하지 않음 |
| `#[IsDestructive]` | bool | 파괴적 갱신을 수행할 수 있음 (`IsReadOnly`가 아닐 때만 의미 있음) |
| `#[IsIdempotent]` | bool | 같은 인자로 반복 호출해도 추가 효과 없음 (`IsReadOnly`가 아닐 때만 의미 있음) |
| `#[IsOpenWorld]` | bool | 외부 엔티티와 상호작용할 수 있음 |

강제 규칙이 아니라 AI 클라이언트에 주는 힌트지만, 위험한 도구(주문 취소, 결제 등)를 AI가 가볍게 반복 호출하는 것을 막는 데 실질적으로 도움이 된다. **파괴적 동작에는 반드시 표시할 것.**

### 4.8 조건부 등록 — `shouldRegister()`

```php
public function shouldRegister(Request $request): bool
{
    return $request?->user()?->subscribed() ?? false;
}
```

`false`를 반환하면 이 Tool은 AI 클라이언트의 목록에서 아예 보이지 않고 호출도 불가능하다 — 구독자 전용 기능, 관리자 전용 기능(주문 취소 등)에 사용.

### 4.9 응답 종류

Tool은 반드시 `Laravel\Mcp\Response` 인스턴스를 반환해야 한다.

```php
return Response::text('Weather Summary: Sunny, 72°F');

return Response::error('Unable to fetch weather data. Please try again.');

return Response::image(file_get_contents(storage_path('weather/radar.png')), 'image/png');
return Response::audio(file_get_contents(storage_path('weather/alert.mp3')), 'audio/mp3');

// 라라벨 파일시스템 디스크에서 바로 로드 (MIME 타입 자동 감지)
return Response::fromStorage('weather/radar.png');
return Response::fromStorage('weather/radar.png', disk: 's3');
return Response::fromStorage('weather/radar.png', mimeType: 'image/webp');
```

**여러 응답 반환** — `Response` 배열을 반환:

```php
/** @return array<int, \Laravel\Mcp\Response> */
public function handle(Request $request): array
{
    return [
        Response::text('Weather Summary: Sunny, 72°F'),
        Response::text('**Detailed Forecast**\n- Morning: 65°F\n- Afternoon: 78°F\n- Evening: 70°F'),
    ];
}
```

**구조화 응답(Structured content)** — AI 클라이언트가 파싱 가능한 데이터를 텍스트와 함께 제공:

```php
return Response::structured([
    'temperature' => 22.5,
    'conditions' => 'Partly cloudy',
    'humidity' => 65,
]);

// 커스텀 텍스트 + 구조화 데이터를 함께
return Response::make(
    Response::text('Weather is 22.5°C and sunny')
)->withStructuredContent([
    'temperature' => 22.5,
    'conditions' => 'Sunny',
]);
```

**스트리밍 응답** — 오래 걸리는 작업의 중간 진행 상황을 실시간으로 보낼 때, `handle()`에서 `Generator`를 반환한다. Web 서버에서는 자동으로 SSE(Server-Sent Events) 스트림이 열린다.

```php
use Generator;

/** @return \Generator<int, \Laravel\Mcp\Response> */
public function handle(Request $request): Generator
{
    $locations = $request->array('locations');

    foreach ($locations as $index => $location) {
        yield Response::notification('processing/progress', [
            'current' => $index + 1,
            'total' => count($locations),
            'location' => $location,
        ]);

        yield Response::text($this->forecastFor($location));
    }
}
```

---

## 5. Prompt — 재사용 가능한 대화 템플릿

```bash
php artisan make:mcp-prompt DescribeWeatherPrompt
```

```php
protected array $prompts = [
    DescribeWeatherPrompt::class,
];
```

이름/제목/설명 커스터마이징은 Tool과 동일 (`#[Name]`, `#[Title]`, `#[Description]`).

### 5.1 인자 정의 — `arguments()` + `Argument`

```php
use Laravel\Mcp\Server\Prompt;
use Laravel\Mcp\Server\Prompts\Argument;

class DescribeWeatherPrompt extends Prompt
{
    /** @return array<int, \Laravel\Mcp\Server\Prompts\Argument> */
    public function arguments(): array
    {
        return [
            new Argument(
                name: 'tone',
                description: 'The tone to use in the weather description (e.g., formal, casual, humorous).',
                required: true,
            ),
        ];
    }
}
```

### 5.2 검증 / 의존성 주입 / 조건부 등록

Tool과 완전히 동일한 패턴 — `handle()`에서 `$request->validate()`, 생성자·`handle()` 파라미터 DI, `shouldRegister()`.

### 5.3 응답 — 시스템 메시지와 사용자 메시지 구분

```php
/** @return array<int, \Laravel\Mcp\Response> */
public function handle(Request $request): array
{
    $tone = $request->string('tone');

    $systemMessage = "You are a helpful weather assistant. Please provide a weather description in a {$tone} tone.";
    $userMessage = "What is the current weather like in New York City?";

    return [
        Response::text($systemMessage)->asAssistant(), // AI 쪽 발화(시스템 프롬프트 역할)로 취급
        Response::text($userMessage),                   // 일반 Response::text()는 사용자 발화로 취급
    ];
}
```

---

## 6. Resource — AI가 "읽기만" 하는 데이터

```bash
php artisan make:mcp-resource WeatherGuidelinesResource
```

```php
protected array $resources = [
    WeatherGuidelinesResource::class,
];
```

기본 URI는 이름 기반 자동 생성(`WeatherGuidelinesResource` → `weather://resources/weather-guidelines`), 기본 MIME 타입은 `text/plain`. `#[Uri]`/`#[MimeType]`으로 커스터마이징 가능.

```php
use Laravel\Mcp\Server\Attributes\MimeType;
use Laravel\Mcp\Server\Attributes\Uri;

#[Uri('weather://resources/guidelines')]
#[MimeType('application/pdf')]
class WeatherGuidelinesResource extends Resource {}
```

### 6.1 Resource Template — URI 패턴 변수로 여러 리소스를 한 클래스로

```php
use Laravel\Mcp\Server\Contracts\HasUriTemplate;
use Laravel\Mcp\Support\UriTemplate;

#[Description('Access user files by ID')]
#[MimeType('text/plain')]
class UserFileResource extends Resource implements HasUriTemplate
{
    public function uriTemplate(): UriTemplate
    {
        return new UriTemplate('file://users/{userId}/files/{fileId}');
    }

    public function handle(Request $request): Response
    {
        $userId = $request->get('userId'); // 템플릿 변수는 자동으로 request에 병합됨
        $fileId = $request->get('fileId');
        $uri = $request->uri();             // 원본 URI 전체도 접근 가능

        // Fetch and return the file content...
        return Response::text($content);
    }
}
```

`HasUriTemplate`를 구현하면 정적 리소스가 아니라 **템플릿**으로 등록되어, URI 패턴에 매칭되는 모든 요청을 하나의 클래스가 처리한다.

### 6.2 Resource Request — 입력 스키마 없음

Tool/Prompt와 달리 Resource는 입력 스키마나 인자를 정의할 수 없다. 다만 `handle()`에서 `Request` 객체를 통해 URI 템플릿 변수 등은 여전히 접근 가능하다.

### 6.3 의존성 주입 / 조건부 등록

Tool과 동일한 패턴 (생성자/`handle()` DI, `shouldRegister()`).

### 6.4 어노테이션

```php
use Laravel\Mcp\Enums\Role;
use Laravel\Mcp\Server\Annotations\Audience;
use Laravel\Mcp\Server\Annotations\LastModified;
use Laravel\Mcp\Server\Annotations\Priority;

#[Audience(Role::User)]             // 대상 독자: Role::User, Role::Assistant, 또는 배열로 둘 다
#[LastModified('2025-01-12T15:00:58Z')]
#[Priority(0.9)]                    // 0.0~1.0, 리소스 중요도
class UserDashboardResource extends Resource {}
```

### 6.5 응답 — text / blob / error

```php
return Response::text($weatherData);

// blob 응답 — MIME 타입은 리소스에 설정된 #[MimeType]을 따름
return Response::blob(file_get_contents(storage_path('weather/radar.png')));

return Response::error('Unable to fetch weather data for the specified location.');
```

---

## 7. 메타데이터(`_meta`)

[MCP 스펙](https://modelcontextprotocol.io/specification/2025-06-18/basic#meta)의 `_meta` 필드를 지원한다 — 일부 MCP 클라이언트/통합에서 요구됨.

```php
// 개별 응답 콘텐츠에 메타데이터
return Response::text('The weather is sunny.')
    ->withMeta(['source' => 'weather-api', 'cached' => true]);

// 응답 envelope 전체에 메타데이터 (Response::make로 감싸기)
use Laravel\Mcp\ResponseFactory;

public function handle(Request $request): ResponseFactory
{
    return Response::make(
        Response::text('The weather is sunny.')
    )->withMeta(['request_id' => '12345']);
}

// Tool/Resource/Prompt 클래스 자체에 메타데이터
class CurrentWeatherTool extends Tool
{
    protected ?array $meta = [
        'version' => '2.0',
        'author' => 'Weather Team',
    ];
}
```

---

## 8. 인증(Authentication)

일반 라우트처럼 **미들웨어**로 Web MCP 서버를 보호한다. 인증을 걸면 서버의 어떤 기능이든 사용 전에 인증이 필요해진다.

### 8.1 OAuth 2.1 — Laravel Passport (가장 권장되는 방식)

MCP 스펙에서 공식 문서화된 인증 방식이며 대부분의 MCP 클라이언트가 지원한다.

```php
use Laravel\Mcp\Facades\Mcp;

Mcp::oauthRoutes(); // OAuth2 discovery / 클라이언트 등록 라우트 등록

Mcp::web('/mcp/weather', WeatherExample::class)
    ->middleware('auth:api');
```

**Passport를 처음 설치하는 경우**, [Passport 설치 가이드](https://laravel.com/docs/12.x/passport#installation)를 먼저 완료(`OAuthenticatable` 모델, 인증 가드, Passport 키 필요)한 뒤:

```bash
php artisan vendor:publish --tag=mcp-views
```

```php
// AppServiceProvider::boot()
use Laravel\Passport\Passport;

public function boot(): void
{
    Passport::authorizationView(function ($parameters) {
        return view('mcp.authorize', $parameters);
    });
}
```

이 뷰는 AI 에이전트의 인증 요청을 사용자가 승인/거부하는 화면이다. (여기선 OAuth를 "기존 인증 가능 모델로의 번역 계층"으로만 쓰며, scope 같은 세부 기능은 다루지 않는다.)

**이미 Passport를 쓰는 경우** — 대부분 그대로 동작하지만 커스텀 scope는 지원되지 않는다. `Mcp::oauthRoutes()`가 단일 `mcp:use` scope를 추가·광고·사용한다.

**Passport vs Sanctum 선택 기준**: OAuth 2.1이 MCP 클라이언트 대부분에서 가장 널리 지원되므로 가능하면 Passport 권장. 이미 Sanctum을 쓰는 앱에 Passport를 추가하는 게 번거롭다면, OAuth 전용 클라이언트가 필요해질 때까지는 Sanctum만으로 미뤄도 된다.

### 8.2 토큰 인증 — Laravel Sanctum

```php
Mcp::web('/mcp/demo', WeatherExample::class)
    ->middleware('auth:sanctum');
```

MCP 클라이언트가 `Authorization: Bearer <token>` 헤더를 보내야 한다.

### 8.3 커스텀 인증

자체 API 토큰 체계가 있다면 임의의 미들웨어를 `Mcp::web` 라우트에 붙여 `Authorization` 헤더를 직접 검사하면 된다.

---

## 9. 인가(Authorization)

`$request->user()`로 현재 인증된 사용자를 가져와 라라벨의 표준 [인가 체크](https://laravel.com/docs/12.x/authorization)를 그대로 쓴다.

```php
public function handle(Request $request): Response
{
    if (! $request->user()->can('read-weather')) {
        return Response::error('Permission denied.');
    }

    // ...
}
```

---

## 10. 테스트

### 10.1 MCP Inspector — 대화형 디버깅 툴

```bash
php artisan mcp:inspector mcp/weather   # Web 서버
php artisan mcp:inspector weather       # Local 서버 (등록한 이름)
```

브라우저 기반 UI와, 실제 MCP 클라이언트에 그대로 붙여넣을 접속 설정값을 함께 보여준다. 인증 미들웨어가 걸려 있으면 Authorization 헤더 등을 Inspector에 직접 입력해서 확인한다.

### 10.2 유닛 테스트 (Pest / PHPUnit)

```php
// Pest
test('tool', function () {
    $response = WeatherServer::tool(CurrentWeatherTool::class, [
        'location' => 'New York City',
        'units' => 'fahrenheit',
    ]);

    $response
        ->assertOk()
        ->assertSee('The current weather in New York City is 72°F and sunny.');
});
```

```php
// PHPUnit
public function test_tool(): void
{
    $response = WeatherServer::tool(CurrentWeatherTool::class, [
        'location' => 'New York City',
        'units' => 'fahrenheit',
    ]);

    $response->assertOk()->assertSee('The current weather in New York City is 72°F and sunny.');
}
```

Prompt/Resource도 동일한 방식:

```php
$response = WeatherServer::prompt(...);
$response = WeatherServer::resource(...);
```

인증된 사용자로 테스트하려면 `actingAs()`를 체이닝:

```php
$response = WeatherServer::actingAs($user)->tool(...);
```

**사용 가능한 어설션**:

```php
$response->assertOk();                      // 에러 없음
$response->assertSee('...');                // 특정 텍스트 포함
$response->assertHasErrors();               // 에러 포함
$response->assertHasErrors(['Something went wrong.']);
$response->assertHasNoErrors();
$response->assertName('current-weather');
$response->assertTitle('Current Weather Tool');
$response->assertDescription('...');
$response->assertSentNotification('processing/progress', ['step' => 1, 'total' => 5]); // 스트리밍 알림 확인
$response->assertNotificationCount(5);
$response->dd();    // 디버깅용 raw 응답 출력 후 종료
$response->dump();  // 디버깅용 raw 응답 출력
```

---

## 11. Claude Code를 위한 실전 체크리스트

새 Tool/Resource/Prompt를 만들 때 순서대로 확인:

1. `make:mcp-*`로 생성했는가?
2. **서버의 `$tools`/`$resources`/`$prompts` 배열에 등록했는가?** (가장 흔히 빠뜨리는 단계)
3. `#[Description]`을 구체적으로 작성했는가? (AI의 호출 판단 근거)
4. Tool이라면: 입력 스키마(`schema()`)를 정의했는가? 검증(`$request->validate()`)이 필요한가?
5. Tool이 데이터를 변경/삭제한다면: `#[IsReadOnly(false)]` + `#[IsDestructive(true)]`를 붙였는가?
6. 인증이 필요한 서버인가? `routes/ai.php`에 `auth:sanctum` 또는 `auth:api` 미들웨어가 붙어 있는가?
7. 사용자별로 접근을 제한해야 하는가? `handle()` 안에서 `$request->user()->can(...)`을 체크했는가, 또는 `shouldRegister()`로 아예 노출을 막아야 하는가?
8. `php artisan mcp:inspector <name>`으로 실제 동작을 확인했는가?

**흔한 실수**:
- Tool/Resource/Prompt 클래스만 만들고 서버에 등록을 빼먹음 → AI 클라이언트에 전혀 보이지 않음
- `#[Description]`을 생략하거나 모호하게 씀 → AI가 도구를 잘못 호출하거나 아예 안 씀
- 파괴적 동작(`CancelOrderTool` 등)에 인가 체크나 `#[IsDestructive]` 표시를 빠뜨림
- `Mcp::web()`에 인증/스로틀 미들웨어 없이 배포 — 원격 클라이언트가 붙는 엔드포인트이므로 일반 API와 동일한 보안 기준 필요
- `handle()`에 도메인 로직을 직접 작성 — 컨트롤러와 마찬가지로 얇게 유지하고 Repository/Service에 위임하는 편이 REST API와 로직을 재사용하기 좋음

---

## 참고

- [Laravel MCP 공식 문서 (13.x, 최신)](https://laravel.com/docs/13.x/mcp)
- [Laravel MCP 공식 문서 (12.x, 이 문서의 원본)](https://laravel.com/docs/12.x/mcp)
- [Introducing Laravel MCP 블로그 포스트](https://laravel.com/blog/introducing-laravel-mcp-build-with-the-universal-ai-standard)
- [Model Context Protocol 공식 사이트](https://modelcontextprotocol.io/docs/getting-started/intro)
- [Laravel MCP GitHub](https://github.com/laravel/mcp)
- [Locket 데모 앱](https://github.com/laravel/locket) ([라이브](https://locket.laravel.cloud))
- [Laravel Boost](https://blog.laravel.com/announcing-laravel-boost) — 이 패키지로 구현된 실전 사례

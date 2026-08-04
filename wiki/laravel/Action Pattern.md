---
title: Action Pattern
category: laravel
tags: [laravel, action, single-responsibility]
related: [[Application Service]], [[DTO]], [[CQRS]]
---

# Action Pattern

하나의 유스케이스를 담당하는 단일 책임 클래스. [[Application Service]]의 단순화 버전. Laravel 커뮤니티(특히 Laravel Actions 패키지)에서 널리 사용된다.

## 핵심 개념

- 클래스 하나 = 유스케이스 하나
- `execute()` 또는 `handle()` 메서드 하나
- HTTP, Queue, CLI 등 다양한 진입점에서 재사용 가능

## 기본 구조

```php
// Application/Action/ConfirmOrderAction.php
final class ConfirmOrderAction
{
    public function __construct(
        private readonly OrderRepository $orders,
        private readonly Mailer $mailer,
    ) {}

    public function execute(OrderId $orderId): void
    {
        $order = $this->orders->findById($orderId)
            ?? throw new OrderNotFoundException($orderId);

        $order->confirm();

        $this->orders->save($order);
    }
}
```

## Laravel Actions 패키지 활용

`lorisleiva/laravel-actions` 패키지를 사용하면 동일 Action을 HTTP, Job, Artisan Command로 실행할 수 있다.

```php
use Lorisleiva\Actions\Concerns\AsAction;

class ConfirmOrder
{
    use AsAction;

    public string $commandSignature = 'order:confirm {orderId}';

    public function handle(OrderId $orderId): void
    {
        // 핵심 로직
    }

    // HTTP Controller로 사용
    public function asController(Request $request): JsonResponse
    {
        $this->handle(OrderId::from($request->route('orderId')));
        return response()->json(['message' => '주문이 확정되었습니다.']);
    }

    // Artisan Command로 사용
    public function asCommand(Command $command): void
    {
        $this->handle(OrderId::from($command->argument('orderId')));
        $command->info('주문 확정 완료');
    }
}
```

## Application Service vs Action

| | Application Service | Action |
|--|--------------------|----|
| 유스케이스 수 | 여러 개 | 하나 |
| 구성 복잡도 | 높음 | 낮음 |
| 재사용성 | 높음 | 높음 |
| 파일 수 | 적음 | 많음 |

소규모~중간 규모: Action Pattern이 심플하고 직관적.  
대규모: Application Service로 관련 유스케이스를 그룹화.

## Action을 작성하는 세 가지 방식

『Domain-Driven Design with Laravel』(Martin Joo)이 정리한 세 가지 스타일:

| 방식 | 예시 | 장단점 |
|------|------|--------|
| 인스턴스 `execute()` | `(new CreateTodoAction)->execute($data)` | 가장 무난. 생성자 주입으로 목킹이 쉬움 |
| `__invoke()` (호출 가능 클래스) | `$createTodoAction($data)` | 다른 클래스와 구분이 확실하지만, 다른 Action 안에서 호출할 땐 `($this->action)($data)`처럼 괄호가 필요해 가독성이 떨어짐 |
| 정적 `execute()` | `CreateTodoAction::execute($data)` | 가장 깔끔한 호출부. 단, 테스트에서 Action을 목킹하기 어렵다 — 인프라(외부 API 등)만 목킹하고 Action 자체는 실제로 실행하는 테스트 전략과 궁합이 좋음 |

세 방식 모두 정답은 없다. 프로젝트 전체에서 하나로 통일하는 것이 중요하다.

## Command/Job은 얇은 디스패처로

Console Command나 Queue Job에서 Action을 호출할 때는, Command/Job 자체에는 로직을 두지 않고 Action 호출만 남긴다.

```php
class ImportSubscribersCommand extends Command
{
    protected $signature = 'subscriber:import {user? : The ID of the user}';

    public function handle()
    {
        $userId = $this->argument('user') ?? $this->ask('User ID');

        ImportSubscribersJob::dispatch(
            storage_path('subscribers/subscribers.csv'),
            User::findOrFail($userId),
        );

        $this->info('Subscribers are being imported...');
    }
}

class ImportSubscribersJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        private readonly string $path,
        private readonly User $user,
    ) {}

    public function handle()
    {
        ImportSubscribersAction::execute($this->path, $this->user);
    }
}
```

이렇게 하면 웹 컨트롤러, Console Command, Queue Job이 모두 같은 `ImportSubscribersAction`을 호출하는 얇은 진입점이 되어, 실제 로직은 한 곳에만 존재한다.

## 어떤 모델에 속하는 쿼리인지 애매할 때

두 모델에 걸친 조회(예: "특정 사용자의 30일 지난 게시물 목록")를 어느 모델에 둘지 애매한 경우가 흔하다.

```php
// 옵션 1: User에 두기 — 사용법은 좋지만 User가 Listing을 알아야 함
class User { public function getOldListings(): Collection { /* Listing 쿼리 */ } }

// 옵션 2: Listing에 두기 — 사용법이 어색함 (Listing::getOldListings($user))
class Listing { public static function getOldListings(User $user) { /* ... */ } }
```

둘 다 완벽하지 않다. Action Pattern은 이 딜레마 자체를 없앤다 — 애초에 어느 모델에도 속하지 않는 별도 클래스로 만든다.

```php
class GetOldListingsByUserAction
{
    public function __invoke(User $user): Collection
    {
        return $user->listings()->published()->accepted()
            ->where('publish_at', '<', now()->subMonth())
            ->get();
    }
}
```

"이 쿼리가 User 모델에 속해야 하나, Listing 모델에 속해야 하나?"라는 질문 자체가 Action Pattern에서는 발생하지 않는다.

## 언제 Action을 안 써도 되는가

- **1인 개발 초기 SaaS / 인디 해킹 단계**: Action은커녕 Service도 필요 없다. 컨트롤러에 바로 작성하고 출시부터 한다.
- **작은 프로젝트**: 모델 수가 적고 CRUD 위주라면 Action 도입이 오히려 과할 수 있다. "작다"의 기준은 프로젝트마다 다르므로 팀이 직접 판단한다.
- **재사용이 중요하지 않은 경우**: Console Command나 Job 없이 단순 API 하나만 있다면 컨트롤러+모델만으로 충분할 수 있다.

## 주의사항

- **Action이 다른 Action 호출**: 허용되지만 깊은 체인은 디버깅을 어렵게 만든다.
- **순환 의존**: Action A가 Action B를 호출하고 Action B가 다시 Action A를 호출하는 경우가 드물게 발생한다. 이런 순환이 생기면 대개 기술적 문제가 아니라 하위 비즈니스 로직 설계에 문제가 있다는 신호다.
- **클래스 수 폭증**: `PublishListingAction`, `PublishListingsJob`, `UnpublishListingAction`처럼 비슷한 이름의 작은 클래스가 급격히 늘어난다. 네임스페이스를 잘 정리하지 않으면 오히려 탐색이 어려워질 수 있다.
- **HTTP 로직 Action에 포함**: Request 파싱이나 Response 생성을 Action에 두면 단일 책임이 깨진다.
- 리소스 컨트롤러(7개 기본 메서드)와 단일 액션(invokable) 컨트롤러를 섞어 쓰는 것은 자연스럽다. 모든 커스텀 동작을 리소스 컨트롤러에 욱여넣거나, 반대로 모든 것을 invokable 컨트롤러로 쪼개면 오히려 탐색이 어려워진다.

## 참고

- [[Application Service]] — 여러 유스케이스를 묶는 서비스
- [[CQRS]] — Command/Query를 명확히 분리하는 패턴
- [[Design Philosophy]] — "정답은 없다, 일관성이 중요하다"는 실용주의 원칙
- 소스: Domain-Driven Design with Laravel (Martin Joo), Actions / Building an E-mail Marketing Software 챕터
- 소스: Layered Architectures with Laravel (Martin Joo)

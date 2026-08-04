---
title: Testing Complex Features (블랙박스 통합 테스트)
category: testing
tags: [testing, integration-test, feature-test, time-travel, queue-fake]
related: [[Feature Testing]], [[Domain Testing]], [[Action Pattern]]
---

# Testing Complex Features (블랙박스 통합 테스트)

CSV 임포트, 이벤트 기반 자동화, 시간 의존적 스케줄링처럼 표준 CRUD를 벗어난 복잡한 기능을 목(mock) 없이 블랙박스로 검증하는 테스트 전략.

## 핵심 개념

저자(Martin Joo)의 기본 원칙: **가능하면 목을 쓰지 않고, 실제로 전부 실행되게 한다.** 이유는 세 가지다.

- **엔드투엔드 검증**: 유스케이스가 A부터 Z까지 실제로 동작하는지 확인한다.
- **작성이 쉽다**: 복잡한 목/스파이/페이크를 준비하는 데 시간을 쓰지 않는다.
- **리팩터링에 강하다**: 블랙박스 테스트이므로 내부 구현을 자유롭게 리팩터링해도 테스트가 깨지지 않는다.

단, 예외도 있다: 하나의 클래스 안에 분기(if/switch)가 아주 많은 경우(예: 9가지 할인 정책이 얽힌 가격 계산 로직)는 고립된 단위 테스트가 오히려 우월할 수 있다.

## Laravel 구현

### 1. 프로덕션 클래스로 테스트 데이터 준비

테스트에서 복잡한 데이터 구조(예: 여러 단계를 가진 Automation)를 만들어야 할 때, 팩토리를 억지로 조합하기보다 **실제 프로덕션 Action을 그대로 호출**해 데이터를 준비한다.

```php
/** @test */
public function it_should_add_tags()
{
    $user = User::factory()->create();
    $form = Form::factory()->create();
    $laravel = Tag::factory(['title' => 'Laravel'])->create();

    // 프로덕션 코드에서 쓰는 Action을 테스트 셋업에도 그대로 사용
    UpsertAutomationAction::execute(
        AutomationData::from([
            'name' => 'Test Automation',
            'event' => new AutomationStepData(null, Events::SubscribedToForm->value, $form->id),
            'actions' => [
                new AutomationStepData(null, Actions::AddTag->value, $laravel->id),
            ],
        ]),
        $user
    );

    $subscriber = UpsertSubscriberAction::execute(/* ... */);

    $this->assertTrue($subscriber->tags()->where('tags.id', $laravel->id)->exists());
}
```

### 2. 이벤트/큐를 동기 실행으로 전환

이벤트 리스너가 Job을 큐에 넣고, 그 Job이 다시 Action을 호출하는 구조라도 `phpunit.xml`에서 큐 드라이버를 `sync`로 설정하면 별도 목킹 없이 체인 전체가 그대로 동기 실행된다.

```xml
<php>
    <server name="QUEUE_CONNECTION" value="sync"/>
    <server name="MAIL_MAILER" value="array"/>
</php>
```

### 3. "아무 일도 일어나지 않았음"을 검증하기

부정 조건(예: 조건이 안 맞으면 자동화가 실행되지 않아야 함)은 `Queue::fake()` + `assertNothingPushed()`로 검증한다.

```php
Queue::fake();
UpsertSubscriberAction::execute(/* form이 null인 데이터 */);
Queue::assertNothingPushed();
```

### 4. 메일 발송 검증

```php
Mail::fake(); // setUp()에서 한 번만 호출해두면 모든 테스트에 적용됨

ProceedSequenceAction::execute($sequence);

$this->assertDatabaseHas('sent_mails', [
    'sendable_id' => $mail->id,
    'subscriber_id' => $subscriber->id,
]);
Mail::assertQueued(EchoMail::class, fn (EchoMail $echoMail) =>
    $echoMail->mail->id() === $mail->id
);
```

### 5. 시간 이동(Time Travel)으로 스케줄 로직 검증

`$this->travelTo()`는 내부적으로 `Carbon::setTestNow()`를 호출해, 콜백 안에서 `now()`가 지정된 시각을 반환하게 만든다. "1시간 뒤 두 번째 메일이 나가야 한다" 같은 지연 스케줄 로직을 실제 시간 흐름 없이 검증할 수 있다.

```php
ProceedSequenceAction::execute($sequence);
$this->assertMailSent($mail1, $subscriber);

$this->travelTo(now()->addHours(2), function () use ($sequence, $mail2, $subscriber) {
    ProceedSequenceAction::execute($sequence);
    $this->assertMailSent($mail2, $subscriber);
    $this->assertCompleted($sequence, $subscriber);
});
```

요일 조건처럼 특정 날짜가 필요하면 `$this->travelTo('2022-03-30', ...)`처럼 고정된 과거 날짜를 문자열로 지정하거나, `$this->travelTo('next friday', ...)`처럼 상대 표현을 쓸 수 있다.

### 6. 반복되는 어서션은 헬퍼 메서드로 추출

`assertTags()`, `assertMailSent()`, `assertMailNotSent()`, `assertInProgress()`, `assertCompleted()`처럼 테스트 전반에서 반복되는 검증 로직은 `TestCase`의 private 헬퍼 메서드로 뽑아내 테스트 본문을 읽기 쉽게 유지한다.

## DDD 관점에서의 활용

이 전략은 [[Action Pattern]]과 궁합이 특히 좋다 — 유스케이스 전체가 하나의 Action `execute()` 호출로 캡슐화되어 있으므로, 테스트도 "Action을 호출하고 결과를 확인한다"는 동일한 형태를 유지할 수 있다. 반대로 로직이 여러 서비스/컨트롤러에 흩어져 있으면 이런 블랙박스 테스트를 작성하기 어려워진다.

## 주의사항 / 안티패턴

- 매직 넘버(CSV의 행 수 등)를 테스트에 하드코딩하지 말고 상수로 추출해 의도를 드러낸다.
- 부정 조건 테스트에서 `Queue::fake()` 호출을 빠뜨리면, 실제로 큐에 무언가 쌓였는지 검증할 수 없어 거짓 양성(false positive)이 발생한다.
- 100% 커버리지를 목표로 모든 분기를 테스트하려 하지 말 것 — 사용자 스토리 단위로 가장 중요한 시나리오를 우선한다.

## 참고

- [[Feature Testing]] — HTTP 요청 단위의 엔드투엔드 테스트
- [[Domain Testing]] — 분기가 많은 로직에 대한 고립된 단위 테스트
- [[Action Pattern]] — 블랙박스 테스트와 잘 맞는 유스케이스 캡슐화
- 소스: Testing Complex Features (Martin Joo)

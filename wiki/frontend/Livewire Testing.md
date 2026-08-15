---
title: Livewire Testing
category: frontend
tags: [frontend, livewire, laravel, testing]
related: [[Livewire Forms and Validation]], [[Livewire Events]], [[Domain Testing]], [[Feature Testing]]
---

# Livewire Testing

Livewire는 브라우저 없이 컴포넌트 동작을 그대로 검증할 수 있다.

## 핵심 개념

```php
use Livewire\Livewire;

it('게시글을 생성한다', function () {
    $user = User::factory()->create();

    Livewire::actingAs($user)
        ->test('post.create')
        ->set('title', '테스트 제목')
        ->set('content', '본문 내용입니다. 충분히 길게.')
        ->call('save')
        ->assertHasNoErrors()
        ->assertRedirect('/posts');

    expect(Post::where('title', '테스트 제목')->exists())->toBeTrue();
});

it('제목이 없으면 검증에 실패한다', function () {
    Livewire::test('post.create')
        ->set('title', '')
        ->call('save')
        ->assertHasErrors(['title' => 'required']);
});

it('검색이 동작한다', function () {
    Post::factory()->create(['title' => 'Laravel 입문']);
    Post::factory()->create(['title' => 'Python 입문']);

    Livewire::test('post.index')
        ->set('search', 'Laravel')
        ->assertSee('Laravel 입문')
        ->assertDontSee('Python 입문');
});
```

### 주요 assertion

```php
Livewire::test('counter')
    ->assertSet('count', 0)
    ->assertNotSet('count', 5)
    ->call('increment')
    ->assertSet('count', 1)
    ->assertSee('1')
    ->assertDontSee('99')
    ->assertSeeHtml('<span>1</span>')
    ->assertCount('items', 3)
    ->assertDispatched('post-created')
    ->assertNotDispatched('error')
    ->assertHasErrors('title')
    ->assertHasErrors(['title' => 'required'])
    ->assertHasNoErrors()
    ->assertRedirect('/posts')
    ->assertNoRedirect()
    ->assertStatus(200)
    ->assertForbidden()
    ->assertViewHas('posts')
    ->assertFileDownloaded('report.pdf');
```

## Laravel 구현

### 파라미터와 모델 전달

```php
$post = Post::factory()->create();

Livewire::test('post.edit', ['post' => $post])
    ->assertSet('title', $post->title)
    ->set('title', '수정된 제목')
    ->call('save');

expect($post->fresh()->title)->toBe('수정된 제목');
```

### 파일 업로드 테스트

```php
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

it('사진을 업로드한다', function () {
    Storage::fake('public');
    $file = UploadedFile::fake()->image('photo.jpg');

    Livewire::test('photo.upload')
        ->set('photo', $file)
        ->call('save')
        ->assertHasNoErrors();

    Storage::disk('public')->assertExists('photos/' . $file->hashName());
});
```

### 이벤트 테스트

```php
it('생성 후 이벤트를 발행한다', function () {
    Livewire::test('post.create')
        ->set('title', '제목')
        ->set('content', '내용입니다 충분히 길게')
        ->call('save')
        ->assertDispatched('post-created');
});

it('이벤트를 받아 목록을 갱신한다', function () {
    Livewire::test('post.index')
        ->dispatch('post-created', postId: 1)
        ->assertSee('...');
});
```

### 페이지 컴포넌트 테스트

```php
it('게시글 목록 페이지가 열린다', function () {
    $this->get('/posts')
        ->assertStatus(200)
        ->assertSeeLivewire('pages::post.index');
});
```

## 주의사항 / 안티패턴

- `Livewire::actingAs()`를 빠뜨리고 인증이 필요한 컴포넌트를 테스트하면 `authorize()` 단계에서 예기치 않게 실패한다.
- `assertHasErrors(['title' => 'required'])`처럼 규칙까지 명시하면 검증 규칙 자체가 바뀌었을 때 테스트가 더 정확히 실패를 잡아준다 — 필드명만 확인하는 `assertHasErrors('title')`보다 회귀 탐지력이 높다.
- 파일 업로드 테스트에서 `Storage::fake()`를 빠뜨리면 실제 디스크에 테스트 파일이 쌓인다.

## 참고

- [[Livewire Forms and Validation]] — 검증 로직과 테스트 대상
- [[Livewire Events]] — `assertDispatched`가 검증하는 이벤트 발행
- [[Domain Testing]], [[Feature Testing]] — 이 위키의 일반 Laravel 테스트 전략과의 관계
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

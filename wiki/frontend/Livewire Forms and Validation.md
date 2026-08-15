---
title: Livewire Forms and Validation
category: frontend
tags: [frontend, livewire, laravel, validation, forms]
related: [[Livewire Properties]], [[Livewire Actions]], [[Form Request]], [[Livewire Loading States]]
---

# Livewire Forms and Validation

Laravel의 `validate()`/`@error`/`$errors`가 그대로 동작하는 Livewire 폼. `#[Validate]` 어트리뷰트와 재사용 가능한 Form 객체까지.

## 핵심 개념

```php
new class extends Component {
    public string $title = '';
    public string $content = '';

    public function save()
    {
        $validated = $this->validate([
            'title'   => 'required|min:3|max:255',
            'content' => 'required|min:10',
        ]);

        Post::create($validated);
        $this->reset();
        session()->flash('message', '게시글이 등록되었습니다.');
    }
};
```

```blade
<form wire:submit="save">
    <input type="text" wire:model="title">
    @error('title') <span class="text-red-600">{{ $message }}</span> @enderror
    <button type="submit">저장</button>
</form>
```

Laravel의 `validate()`, `@error`, `$errors`가 **그대로** 동작하는 것이 Livewire의 큰 장점이다 — [[Form Request]]에서 쓰던 지식을 그대로 재사용한다.

### `#[Validate]` 어트리뷰트

규칙을 프로퍼티 옆에 붙일 수 있다.

```php
use Livewire\Attributes\Validate;

#[Validate('required|min:3|max:255')]
public string $title = '';

#[Validate('required|min:3', message: '제목은 3자 이상 입력해 주세요.')]
public string $title = '';

#[Validate('required|email', as: '이메일 주소')]
public string $email = '';

#[Validate([
    'tags'   => 'required|array|min:1',
    'tags.*' => 'string|max:20',
])]
public array $tags = [];

public function save()
{
    $this->validate();   // 규칙을 인자로 넘길 필요 없음
}
```

### 실시간 검증

```blade
<input wire:model.blur="email">   {{-- 포커스를 잃을 때 검증 (권장) --}}
<input wire:model.live="email">   {{-- 타이핑마다 검증 — 요청이 너무 많아 비권장 --}}
```

`wire:model.blur`를 쓰면 값이 서버로 갈 때 `#[Validate]` 규칙이 자동으로 돈다. 특정 필드만 수동 검증하려면:

```php
public function updatedEmail() { $this->validateOnly('email'); }
```

### 커스텀 규칙 메서드

동적 규칙이 필요할 때는 `rules()` 메서드로 오버라이드한다.

```php
protected function rules(): array
{
    return [
        'slug' => ['required', 'alpha_dash', Rule::unique('posts', 'slug')->ignore($this->postId)],
    ];
}

protected function messages(): array
{
    return ['slug.unique' => '이미 사용 중인 슬러그입니다.'];
}
```

## Laravel 구현

### Form 객체 — 재사용 가능한 폼

폼 로직이 커지면 별도 클래스로 분리한다. **생성 폼과 수정 폼이 같은 Form 클래스를 공유**하고, 컴포넌트가 얇아진다(FormRequest와 비슷한 역할 + 상태까지 담당).

```bash
php artisan livewire:form PostForm
# → app/Livewire/Forms/PostForm.php
```

```php
namespace App\Livewire\Forms;

use Livewire\Attributes\Validate;
use Livewire\Form;

class PostForm extends Form
{
    public ?Post $post = null;

    #[Validate('required|min:3|max:255')]
    public string $title = '';

    #[Validate('required|min:10')]
    public string $content = '';

    public function setPost(Post $post): void
    {
        $this->post    = $post;
        $this->title   = $post->title;
        $this->content = $post->content;
    }

    public function store(): Post
    {
        $this->validate();
        $post = Post::create($this->only('title', 'content'));
        $this->reset();
        return $post;
    }

    public function update(): Post
    {
        $this->validate();
        $this->post->update($this->only('title', 'content'));
        return $this->post;
    }
}
```

```php
new class extends Component {
    public PostForm $form;

    public function mount(?Post $post = null)
    {
        if ($post?->exists) $this->form->setPost($post);
    }

    public function save()
    {
        $this->form->post ? $this->form->update() : $this->form->store();
        return $this->redirect('/posts', navigate: true);
    }
};
```

```blade
<form wire:submit="save">
    <input wire:model="form.title">
    @error('form.title') <span class="text-red-600">{{ $message }}</span> @enderror
    <button type="submit">저장</button>
</form>
```

`$this->form->only()`, `$this->form->all()`, `$this->form->reset()`, `$this->form->pull()`도 사용 가능하다.

### 에러 표시 패턴

```blade
@error('title') <span>{{ $message }}</span> @enderror
@error('form.title') <span>{{ $message }}</span> @enderror   {{-- 중첩 --}}
@error('tags.*') <span>{{ $message }}</span> @enderror        {{-- 와일드카드 --}}

@if ($errors->any())
    <ul>@foreach ($errors->all() as $error) <li>{{ $error }}</li> @endforeach</ul>
@endif

<input wire:model="title"
       class="border @error('title') border-red-500 @else border-gray-300 @enderror">
```

JavaScript에서 에러 접근(v4):

```blade
<div wire:show="$errors.has('email')">
    <span wire:text="$errors.first('email')"></span>
</div>
```

### 수동 에러 조작

```php
public function save()
{
    if (! $this->checkExternalApi()) {
        $this->addError('title', '외부 시스템 검증에 실패했습니다.');
        return;
    }

    $this->resetValidation();          // 전체 에러 초기화
    $this->resetValidation('title');   // 특정 필드만
}
```

## 주의사항 / 안티패턴

- `wire:model.live`로 매 타이핑마다 검증하면 요청이 과도하게 늘어난다 — `wire:model.blur`를 기본으로 쓴다.
- `rules()` 메서드를 오버라이드하면서 `#[Validate]`와 동시에 쓰면 어느 쪽이 최종 규칙인지 헷갈릴 수 있다 — 동적 규칙이 필요한 프로퍼티만 `rules()`로 분리하는 편이 낫다.
- 저장 중 중복 제출을 막으려면 버튼에 `wire:loading.attr="disabled"`와 `wire:target`을 함께 건다 (자세한 건 [[Livewire Loading States]]).

## 참고

- [[Livewire Properties]] — 폼 필드가 되는 public 프로퍼티
- [[Livewire Actions]] — `save()` 액션과 파라미터 보안
- [[Form Request]] — Laravel 표준 검증과의 관계
- [[Livewire Loading States]] — 저장 중 중복 제출 방지 UI
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

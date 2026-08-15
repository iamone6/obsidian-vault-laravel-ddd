---
title: Livewire Pagination and File Uploads
category: frontend
tags: [frontend, livewire, laravel]
related: [[Livewire Properties]], [[Livewire URL and Navigation]], [[Livewire and NativePHP]]
---

# Livewire Pagination and File Uploads

목록 화면과 폼에서 실무 빈도가 가장 높은 두 기능 — 페이지네이션과 파일 업로드.

## 핵심 개념

### Pagination

```php
use Livewire\WithPagination;

new class extends Component {
    use WithPagination;

    #[Url]
    public string $search = '';

    public function updatedSearch() { $this->resetPage(); }   // 검색어가 바뀌면 1페이지로

    public function render()
    {
        return $this->view([
            'posts' => Post::query()
                ->when($this->search, fn ($q) => $q->where('title', 'like', "%{$this->search}%"))
                ->paginate(15),
        ]);
    }
};
```

```blade
<div>{{ $posts->links() }}</div>
```

이동 메서드: `$this->setPage(3)`, `$this->nextPage()`, `$this->previousPage()`, `$this->gotoPage(5)`, `$this->resetPage()`.

한 페이지에 목록이 둘 이상이면 쿼리스트링 이름이 충돌하므로 `pageName`을 분리한다.

```php
'posts'    => Post::paginate(10, pageName: 'postsPage'),
'comments' => Comment::paginate(10, pageName: 'commentsPage'),
```

### 커서 페이지네이션

대용량 데이터에서 훨씬 빠르다.

```php
use Livewire\WithoutUrlPagination;

new class extends Component {
    use WithPagination, WithoutUrlPagination;

    public function render()
    {
        return $this->view(['posts' => Post::cursorPaginate(20)]);
    }
};
```

### 무한 스크롤 패턴

```php
new class extends Component {
    public int $perPage = 12;

    public function loadMore() { $this->perPage += 12; }

    #[Computed]
    public function posts() { return Post::latest()->take($this->perPage)->get(); }

    #[Computed]
    public function hasMore() { return Post::count() > $this->perPage; }
};
```

```blade
@if ($this->hasMore)
    {{-- v4: 화면에 들어오면 자동 로드 --}}
    <div wire:intersect="loadMore">
        <span wire:loading wire:target="loadMore">불러오는 중...</span>
    </div>
@endif
```

## Laravel 구현

### File Uploads

```php
use Livewire\WithFileUploads;
use Livewire\Attributes\Validate;

new class extends Component {
    use WithFileUploads;

    #[Validate('image|max:2048')]   // 2MB
    public $photo;

    #[Validate(['photos.*' => 'image|max:2048'])]
    public array $photos = [];

    public function save()
    {
        $this->validate();

        $path = $this->photo->store('photos', 'public');
        // 원본 파일명 유지
        $path = $this->photo->storeAs('photos', $this->photo->getClientOriginalName(), 'public');

        Photo::create(['path' => $path]);
        $this->reset('photo');
    }

    public function removePhoto(int $index)
    {
        unset($this->photos[$index]);
        $this->photos = array_values($this->photos);
    }
};
```

```blade
<form wire:submit="save">
    <input type="file" wire:model="photo" accept="image/*">

    {{-- 업로드 진행률 --}}
    <div x-data="{ progress: 0 }"
         x-on:livewire-upload-progress="progress = $event.detail.progress"
         x-on:livewire-upload-finish="progress = 0"
         x-show="progress > 0">
        <div :style="`width: ${progress}%`"></div>
    </div>

    {{-- 미리보기 --}}
    @if ($photo)
        <img src="{{ $photo->temporaryUrl() }}">
    @endif

    @error('photo') <span>{{ $message }}</span> @enderror
    <button type="submit">업로드</button>
</form>
```

업로드 이벤트(Alpine):

```blade
<div x-data="{ uploading: false, progress: 0 }"
     x-on:livewire-upload-start="uploading = true"
     x-on:livewire-upload-finish="uploading = false"
     x-on:livewire-upload-cancel="uploading = false"
     x-on:livewire-upload-error="uploading = false; alert('업로드 실패')"
     x-on:livewire-upload-progress="progress = $event.detail.progress">
</div>
```

다중 업로드: `<input type="file" wire:model="photos" multiple>` + `@foreach ($photos as $index => $photo)`로 반복.

### 임시 파일 정리

Livewire는 업로드 파일을 `livewire-tmp` 디렉토리에 임시 저장한 뒤 24시간 후 자동 삭제한다. S3를 쓴다면 설정이 필요하다.

```php
// config/livewire.php
'temporary_file_upload' => [
    'disk'       => 's3',
    'rules'      => ['file', 'max:12288'],
    'directory'  => 'tmp',
    'middleware' => 'throttle:60,1',
    'max_upload_time' => 5,
],
```

## 주의사항 / 안티패턴

- 검색/필터가 바뀌었는데 `resetPage()`를 호출하지 않으면 필터링된 결과가 5페이지째부터 시작해 빈 화면이 뜨는 흔한 버그가 생긴다.
- 한 페이지에 페이지네이션이 둘 이상인데 `pageName`을 분리하지 않으면 쿼리스트링이 충돌해 둘 다 같이 페이지가 넘어간다.
- `unset($this->photos[$index])` 후 `array_values()`를 빠뜨리면 [[Livewire Properties]]에서 다룬 배열 인덱스 구멍 문제가 그대로 재현된다.

## 참고

- [[Livewire Properties]] — 배열 다루기, `unset` 후 재정렬 규칙
- [[Livewire URL and Navigation]] — `#[Url]`로 검색/필터 상태를 URL에 반영
- [[Livewire and NativePHP]] — 모바일에서는 페이지네이션보다 무한 스크롤이 흔한 이유
- 소스: `2026-08-12_Livewire 완전 학습 가이드.md`

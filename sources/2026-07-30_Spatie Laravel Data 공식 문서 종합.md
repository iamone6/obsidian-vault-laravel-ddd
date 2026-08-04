# Spatie Laravel Data 개발 레퍼런스 (공식 문서 + 소스 + 블로그 종합)

> **출처**
> - [GitHub - spatie/laravel-data](https://github.com/spatie/laravel-data) (소스 코드, `docs/` 디렉토리 원문)
> - [공식 문서 v4](https://spatie.be/docs/laravel-data/v4/introduction)
> - [byzz 블로그 - Laravel DTO 작성하기](https://byzz.tistory.com/entry/Laravel-DTO-%EC%9E%91%EC%84%B1%ED%95%98%EA%B8%B0)
> - 이 문서는 **기존에 `FormRequest`로 검증을 하던 개발자가 laravel-data의 Attribute 기반 검증으로 넘어갈 때 바로 참고할 수 있도록**, 입문부터 실무 적용(예외 처리 포함)까지 코드 예제 위주로 재구성했습니다.

---

## 0. 한 줄 요약

`spatie/laravel-data`는 **DTO + FormRequest(검증) + API Resource(응답 변환) + TypeScript 타입**을 하나의 `Data` 클래스로 통합해주는 패키지다. "데이터 구조를 한 번만 정의하면 된다"가 핵심 철학이다.

```bash
composer require spatie/laravel-data
```

---

## 1. 설치 및 설정

```bash
composer require spatie/laravel-data

# 설정 파일 publish (캐싱, 검증 전략, 이름 매핑 전략 등을 커스터마이징하고 싶을 때)
php artisan vendor:publish --provider="Spatie\LaravelData\LaravelDataServiceProvider" --tag="data-config"
```

`config/data.php`에서 특히 눈여겨볼 옵션:

```php
// Data 객체는 기본적으로 "요청(Request)으로부터 생성될 때만" 검증된다.
// Always 로 바꾸면 from() 호출 시 항상 검증하고, Disabled 로 바꾸면 검증을 끈다.
'validation_strategy' => \Spatie\LaravelData\Support\Creation\ValidationStrategy::OnlyRequests->value,

// rule inferrer 순서 - 프로퍼티 타입으로부터 검증 규칙을 자동 생성하는 5단계
'rule_inferrers' => [
    Spatie\LaravelData\RuleInferrers\SometimesRuleInferrer::class,
    Spatie\LaravelData\RuleInferrers\NullableRuleInferrer::class,
    Spatie\LaravelData\RuleInferrers\RequiredRuleInferrer::class,
    Spatie\LaravelData\RuleInferrers\BuiltInTypesRuleInferrer::class,
    Spatie\LaravelData\RuleInferrers\AttributesRuleInferrer::class,
],
```

---

## 2. 기본 개념 - Data 클래스 작성

`Spatie\LaravelData\Data`를 상속받고, 생성자 프로퍼티 승격(promoted property)으로 필드를 선언하는 것이 기본 형태다.

```php
<?php

namespace App\Data;

use Spatie\LaravelData\Data;

// Data 를 상속받기만 하면 DTO + Request 검증 + Resource 변환을 모두 지원하는 클래스가 된다.
class SongData extends Data
{
    public function __construct(
        public readonly string $title,   // 문자열, 필수 (readonly 권장 - 불변 객체)
        public readonly string $artist,  // 문자열, 필수
    ) {
    }
}
```

프로퍼티 타입(`string`, `?int`, `bool` 등)만으로 검증 규칙이 자동 추론된다 (7절 참고).

### 데이터 객체 생성 방법

```php
// 배열로부터 생성
$song = SongData::from([
    'title' => 'Never gonna give you up',
    'artist' => 'Rick Astley',
]);

// JSON 문자열로부터 생성
$song = SongData::from('{"title":"Never Gonna Give You Up","artist":"Rick Astley"}');

// Eloquent 모델로부터 생성 (프로퍼티명이 일치하면 자동 매핑)
$song = SongData::from(Song::findOrFail($id));

// 순수 PHP 생성자 호출도 그대로 가능
$song = new SongData('Never gonna give you up', 'Rick Astley');
```

여러 개를 한 번에 만들 때는 `collect()`를 쓴다. 배열, Laravel Collection, 페이지네이터 등 어떤 컬렉션 타입을 넣어도 같은 타입으로 반환된다.

```php
// 배열 -> SongData 배열
$songs = SongData::collect([
    ['title' => 'Never Gonna Give You Up', 'artist' => 'Rick Astley'],
    ['title' => 'Giving Up on Love', 'artist' => 'Rick Astley'],
]);

// Eloquent 컬렉션 -> SongData가 담긴 Eloquent Collection
$songs = SongData::collect(Song::all());

// 페이지네이터 -> SongData가 담긴 LengthAwarePaginator (페이지네이션 메타데이터 유지)
$songs = SongData::collect(Song::paginate());
```

### 마법적 생성 메서드 (`fromXxx`)

특정 타입에 대해 생성 동작을 커스터마이징하고 싶으면 `from`으로 시작하는 정적 메서드를 추가한다 (정적/public, 이름이 `from` 자체는 불가).

```php
class SongData extends Data
{
    public function __construct(
        public string $title,
        public string $artist,
    ) {
    }

    // "제목|아티스트" 형태의 문자열로부터 생성
    public static function fromString(string $string): self
    {
        [$title, $artist] = explode('|', $string);
        return new self($title, $artist);
    }
}
```

---

## 3. 컨트롤러에서 자동 검증 + 생성 (FormRequest 대체)

컨트롤러 메서드 인자로 Data 클래스를 타입힌트하면, FormRequest처럼 자동으로 검증 후 채워진 객체가 주입된다.

```php
<?php

namespace App\Http\Controllers;

use App\Data\SongData;
use App\Models\Song;
use Illuminate\Http\RedirectResponse;

class UpdateSongController
{
    // SongData 를 타입힌트하는 순간 다음이 자동으로 일어난다.
    // 1) SongData 의 프로퍼티 타입 + Validation Attribute 로부터 검증 규칙을 만든다.
    // 2) 현재 Request 를 그 규칙으로 검증한다. 실패하면 ValidationException 발생.
    // 3) 검증을 통과한 값으로 SongData 객체를 만들어 주입한다.
    public function __invoke(Song $model, SongData $data): RedirectResponse
    {
        $model->update($data->all()); // 검증이 끝났으므로 안심하고 사용 가능

        return redirect()->back();
    }
}
```

기존 `FormRequest`를 그대로 두고 싶다면, `Request`(또는 `FormRequest`) 객체를 `from()`에 넘겨도 동일하게 동작한다.

```php
public function __invoke(Song $model, SongRequest $request): RedirectResponse
{
    // FormRequest 를 넘겨도 SongData 의 규칙으로 다시 검증 후 생성된다.
    $model->update(SongData::from($request)->all());

    return redirect()->back();
}
```

컨테이너에서 직접 resolve 할 수도 있다 (요청 값으로 자동 채워짐).

```php
app(SongData::class); // 현재 요청 데이터로 채워진 SongData, 검증 실패 시 ValidationException
```

> ⚠️ **중요**: `Data`는 FormRequest의 `authorize()`에 해당하는 **인가(권한) 기능을 제공하지 않는다.** 공식 문서 어디에도 `authorize` 관련 기능이 없다 (문서 전체 검색 결과 확인). 인가는 여전히 Policy/Gate나 컨트롤러 미들웨어(`can:`)에서 별도로 처리해야 한다.

### 데이터 객체 컬렉션의 검증

```php
class AlbumData extends Data
{
    public function __construct(
        public string $title,
        #[DataCollectionOf(SongData::class)]
        public DataCollection $songs,
    ) {
    }
}
```

`SongData`가 자체 검증 규칙을 가지고 있으므로, `AlbumData`의 규칙을 만들 때 자동으로 병합된다.

```php
[
    'title' => ['required', 'string'],
    'songs' => ['required', 'array'],
    'songs.*.title' => ['required', 'string'],
    'songs.*.artist' => ['required', 'string'],
]
```

---

## 4. 중첩 데이터 / 컬렉션

```php
class ArtistData extends Data
{
    public function __construct(
        public string $name,
        public int $age,
    ) {
    }
}

class AlbumData extends Data
{
    public function __construct(
        public string $title,
        public ArtistData $artist, // 중첩된 Data 객체
    ) {
    }
}
```

```php
// 중첩 배열로부터도 자동으로 재귀 변환된다.
AlbumData::from([
    'title' => 'Whenever You Need Somebody',
    'artist' => [
        'name' => 'Rick Astley',
        'age' => 22,
    ],
]);
```

Data 객체의 컬렉션을 담을 땐, IDE/정적분석기 지원을 위해 반드시 타입을 명시해야 한다 (docblock 또는 제네릭).

```php
use App\Data\SongData;
use Illuminate\Support\Collection;

class AlbumData extends Data
{
    public function __construct(
        public string $title,

        /** @var Collection<int, SongData> 컬렉션에 담긴 데이터 타입을 명시해야 검증/변환이 정확히 동작 */
        public Collection $songs,
    ) {
    }
}
```

이렇게 타입이 명시되면 검증 규칙도 `songs.*.title`, `songs.*.artist` 형태로 자동 생성된다.

---

## 5. Optional / 기본값 / 속성명 매핑

### Optional (부분 업데이트에 유용)

PATCH처럼 일부 필드만 오는 경우, `Optional` 타입을 써서 "값이 아예 전달되지 않음"과 "null"을 구분할 수 있다.

```php
use Spatie\LaravelData\Optional;

class SongData extends Data
{
    public function __construct(
        public string $title,
        public string|Optional $artist, // artist 가 요청에 없으면 Optional 인스턴스가 됨
    ) {
    }
}

// title 만 보내면 artist 는 자동으로 Optional 이 되고,
// toArray() 결과에도 artist 키 자체가 빠진다 (부분 업데이트에 적합).
SongData::from(['title' => '제목만 있는 경우']);
```

### 기본값

```php
class SongData extends Data
{
    public function __construct(
        public string $title,
        public string $genre = 'Pop', // 단순 타입은 생성자에서 바로 기본값 지정
    ) {
    }
}
```

> 기본값이 있는 프로퍼티는 **값이 명시적으로 전달됐을 때만 검증**된다. 기본값 자체는 검증되지 않는다.

### 속성명 매핑 (camelCase ↔ snake_case)

```php
use Spatie\LaravelData\Attributes\MapName;
use Spatie\LaravelData\Mappers\SnakeCaseMapper;

// 클래스 전체에 적용하면 모든 프로퍼티에 규칙이 적용된다.
#[MapName(SnakeCaseMapper::class)] // 입력(snake_case -> camelCase), 출력(camelCase -> snake_case) 모두 적용
class SongData extends Data
{
    public function __construct(
        public string $title,
        public string $releaseDate, // 요청/응답 모두 release_date 로 자동 변환
    ) {
    }
}
```

개별 프로퍼티만 다르게 매핑하고 싶으면:

```php
class SongData extends Data
{
    public function __construct(
        public string $title,

        #[MapInputName('record_company')] // 입력 데이터의 record_company 값을 이 프로퍼티에 매핑
        public string $recordCompany,
    ) {
    }
}
```

---

## 6. 자동 규칙 추론 (Auto Rule Inferring)

Attribute를 하나도 안 붙여도, 프로퍼티의 **PHP 타입**만으로 기본 규칙이 자동 생성된다.

```php
class SongData extends Data
{
    public function __construct(
        public string $title,   // -> ['required', 'string']
        public int $year,       // -> ['required', 'integer']
        public ?string $genre,  // -> ['nullable', 'string']
    ) {
    }
}
```

| Inferrer | 역할 |
|---|---|
| `SometimesRuleInferrer` | Optional 프로퍼티에 `sometimes` 추가 |
| `NullableRuleInferrer` | `?type` 프로퍼티에 `nullable` 추가 |
| `RequiredRuleInferrer` | nullable이 아닌 프로퍼티에 `required` 추가 |
| `BuiltInTypesRuleInferrer` | `string`→`string`, `int/float`→`numeric`, `bool`→`boolean`, `array`→`array` 매핑 |
| `AttributesRuleInferrer` | 아래 7절의 Validation Attribute들을 규칙에 병합 |

---

## 7. Validation Attribute (`FormRequest::rules()` 대체)

`Spatie\LaravelData\Attributes\Validation` 네임스페이스 아래에 **Laravel의 거의 모든 validation 규칙에 대응하는 Attribute 클래스**가 있다 (공식 문서 기준 총 91개).

```php
<?php

namespace App\Data;

use Spatie\LaravelData\Data;
use Spatie\LaravelData\Attributes\Validation\Max;
use Spatie\LaravelData\Attributes\Validation\Min;
use Spatie\LaravelData\Attributes\Validation\Email;
use Spatie\LaravelData\Attributes\Validation\Uuid;
use Spatie\LaravelData\Attributes\Validation\In;
use Spatie\LaravelData\Attributes\Validation\Unique;

class SongData extends Data
{
    public function __construct(
        #[Uuid] // uuid 형식이어야 함 (required, string 은 타입 추론으로 자동 추가됨)
        public string $uuid,

        #[Min(2), Max(100)] // 최소 2자, 최대 100자
        public string $title,

        #[Email] // 이메일 형식
        public string $contactEmail,

        #[In(['pop', 'rock', 'jazz'])] // 허용된 값 중 하나
        public string $genre,

        #[Unique('songs', 'title')] // songs 테이블의 title 컬럼과 중복되면 안 됨
        public string $slug,
    ) {
    }
}
```

동일한 프로퍼티에 여러 Attribute를 콤마로 나열하면 규칙 배열에 합쳐진다. 위 `title`은 최종적으로 다음과 동일하다.

```php
'title' => ['required', 'string', 'min:2', 'max:100'],
```

`required`, `string`을 직접 쓸 필요가 없다 — 타입에서 자동으로 나온다. **Attribute는 "타입만으로는 표현 못 하는 추가 제약"만 선언하면 된다.**

### 자주 쓰는 Attribute 분류표

| 분류 | Attribute | 예시 | Laravel 규칙 대응 |
|---|---|---|---|
| 필수/조건 | `Required`, `Nullable`, `Sometimes` | `#[Sometimes]` | `required`, `nullable`, `sometimes` |
| 문자열 | `StringType`, `Min`, `Max`, `Between`, `Size` | `#[Between(2, 20)]` | `string`, `min:`, `max:`, `between:`, `size:` |
| 형식 | `Email`, `Url`, `Uuid`, `Ulid`, `IP`, `Alpha`, `AlphaNumeric`, `Regex` | `#[Regex('/^[A-Z]+$/')]` | `email`, `url`, `uuid`, `regex:` |
| 숫자 | `IntegerType`, `Numeric`, `Digits`, `DigitsBetween`, `MultipleOf` | `#[Digits(6)]` | `integer`, `numeric`, `digits:` |
| 날짜 | `Date`, `DateFormat`, `After`, `Before`, `AfterOrEqual`, `BeforeOrEqual` | `#[After('today')]` | `date`, `date_format:`, `after:` |
| 목록 | `In`, `NotIn`, `Distinct`, `ArrayType` | `#[In(['a','b'])]` | `in:`, `not_in:`, `distinct` |
| DB 제약 | `Exists`, `Unique` | `#[Exists('users', 'id')]` | `exists:`, `unique:` |
| 파일 | `File`, `Image`, `Mimes`, `MimeTypes`, `Dimensions` | `#[Image, Max(2048)]` | `file`, `image`, `mimes:` |
| 조건부 | `RequiredIf`, `RequiredUnless`, `RequiredWith`, `ProhibitedIf` | `#[RequiredIf('type', 'company')]` | `required_if:`, `prohibited_if:` |
| 커스텀 | `Rule` | `#[Rule('required\|string')]` | 임의 규칙 문자열/배열 직접 지정 |

전체 91개 목록(알파벳 순, 공식 문서 `advanced-usage/validation-attributes.md` 기준): Accepted, AcceptedIf, ActiveUrl, After, AfterOrEqual, Alpha, AlphaDash, AlphaNumeric, ArrayType, Bail, Before, BeforeOrEqual, Between, BooleanType, Confirmed, CurrentPassword, Date, DateEquals, DateFormat, Declined, DeclinedIf, Different, Digits, DigitsBetween, Dimensions, Distinct, DoesntEndWith, DoesntStartWith, Email, EndsWith, Enum, ExcludeIf, ExcludeUnless, ExcludeWith, ExcludeWithout, Exists, File, Filled, GreaterThan, GreaterThanOrEqualTo, Image, In, InArray, IntegerType, IP, IPv4, IPv6, Json, LessThan, LessThanOrEqualTo, Lowercase, ListType, MacAddress, Max, MaxDigits, MimeTypes, Mimes, Min, MinDigits, MultipleOf, NotIn, NotRegex, Nullable, Numeric, Password, Present, Prohibited, ProhibitedIf, ProhibitedUnless, Prohibits, Regex, Required, RequiredIf, RequiredUnless, RequiredWith, RequiredWithAll, RequiredWithout, RequiredWithoutAll, RequiredArrayKeys, Rule, Same, Size, Sometimes, StartsWith, StringType, TimeZone, Unique, Uppercase, Url, Ulid, Uuid.

> Laravel의 `unique`, `regex` 규칙처럼 **`|` 문자를 규칙 구분자로 쓰는 문자열 대신 배열 문법을 우선 사용**하는 것이 공식 권장 사항이다 (정규식에 `|`가 들어갈 수 있기 때문).

---

## 8. 수동 규칙 정의 (`rules()` 메서드)

Attribute로 표현하기 어려운 복잡한 규칙(커스텀 Rule 객체, 다른 서비스 의존성이 필요한 규칙 등)은 정적 `rules()` 메서드로 직접 정의할 수 있다. `FormRequest::rules()`와 거의 동일한 감각이다.

```php
use Illuminate\Validation\Rule;

class SongData extends Data
{
    public function __construct(
        public string $title,
        public string $artist,
    ) {
    }

    public static function rules(): array
    {
        return [
            'title' => ['required', 'string'],
            'artist' => ['required', 'string'],
        ];
    }
}
```

⚠️ `rules()`를 정의한 프로퍼티는 **자동 타입 추론이 꺼진다.** 아래처럼 `max:20`만 쓰면 `required`, `string`은 자동으로 붙지 않는다.

```php
public static function rules(): array
{
    return [
        'title' => ['max:20'], // required, string 자동 추가 안 됨!
    ];
}
```

자동 추론 규칙과 수동 규칙을 **병합**하고 싶다면 클래스에 `#[MergeValidationRules]`를 붙인다.

```php
use Spatie\LaravelData\Attributes\MergeValidationRules;

#[MergeValidationRules]
class SongData extends Data
{
    public function __construct(
        public string $title,
    ) {
    }

    public static function rules(): array
    {
        return [
            'title' => ['max:20'], // 자동 추론된 required, string 과 병합되어 ['required', 'string', 'max:20']
        ];
    }
}
```

`rules()`에는 의존성 주입과 컨텍스트 접근도 가능하다.

```php
use Spatie\LaravelData\Support\Validation\ValidationContext;

class SongData extends Data
{
    public function __construct(
        public string $title,
        public ?string $artist,
    ) {
    }

    // ValidationContext 를 주입받으면 전체(또는 상대) 페이로드에 접근할 수 있다.
    // 매개변수 이름이 반드시 $context 여야 주입된다.
    public static function rules(ValidationContext $context): array
    {
        return [
            'title' => ['required'],
            // title 이 특정 값일 때만 artist 를 필수로 만드는 동적 규칙
            'artist' => Rule::requiredIf($context->payload['title'] !== '무제'),
        ];
    }
}
```

---

## 9. 고급 Attribute 활용

### 라우트 파라미터 참조 (수정 시 자기 자신 unique 무시)

```php
use Spatie\LaravelData\Attributes\Validation\Unique;
use Spatie\LaravelData\Support\Validation\References\RouteParameterReference;

class SongData extends Data
{
    public function __construct(
        public string $title,

        // 현재 라우트의 {song} 파라미터(모델의 id)는 unique 검사에서 제외
        #[Unique('songs', ignore: new RouteParameterReference('song'))]
        public int $id,
    ) {
    }
}
```

### 현재 로그인한 사용자 참조

```php
use Spatie\LaravelData\Support\Validation\References\AuthenticatedUserReference;

class UserData extends Data
{
    public function __construct(
        public string $name,

        // 현재 로그인한 사용자 자신의 이메일은 unique 검사에서 제외 (본인 정보 수정 시 유용)
        #[Unique('users', 'email', ignore: new AuthenticatedUserReference())]
        public string $email,
    ) {
    }
}
```

### 다른 필드 참조 (조건부 필수)

```php
use Spatie\LaravelData\Attributes\Validation\RequiredIf;

class SongData extends Data
{
    public function __construct(
        public string $title,

        // title 이 특정 값일 때만 artist 를 필수로
        #[RequiredIf('title', '무제')]
        public string $artist,
    ) {
    }
}
```

### 데이터베이스 제약 조건 (`where`)

```php
class UserData extends Data
{
    public function __construct(
        #[Exists('users', where: new WhereConstraint('active', true))]
        public int $user_id,

        #[Unique('users', 'email', where: new WhereNullConstraint('deleted_at'))]
        public string $email,
    ) {
    }
}
```

사용 가능한 제약: `WhereConstraint`, `WhereNotConstraint`, `WhereNullConstraint`, `WhereNotNullConstraint`, `WhereInConstraint`, `WhereNotInConstraint`.

### 커스텀 Validation Attribute 만들기

내장 Attribute로 부족하면 `CustomValidationAttribute`를 상속해 직접 만들 수 있다.

```php
use Attribute;
use Spatie\LaravelData\Attributes\Validation\CustomValidationAttribute;
use Spatie\LaravelData\Support\Validation\ValidationPath;

#[Attribute(Attribute::TARGET_PROPERTY | Attribute::TARGET_PARAMETER)]
class KoreanPhoneNumber extends CustomValidationAttribute
{
    // 이 프로퍼티에 적용될 실제 규칙(객체/문자열)을 반환
    public function getRules(ValidationPath $path): array|object|string
    {
        return ['regex:/^01[0-9]-\d{3,4}-\d{4}$/'];
    }
}
```

```php
class UserData extends Data
{
    public function __construct(
        #[KoreanPhoneNumber]
        public string $phone,
    ) {
    }
}
```

> 커스텀 Attribute는 프로퍼티 위에서만 사용 가능하며, `rules()` 메서드 안에서 클래스처럼 재사용할 수는 없다.

---

## 10. 검증 건너뛰기

특정 프로퍼티가 요청 페이로드에 직접 존재하지 않고, 매직 생성 메서드(`fromXxx`)에서 조합되어 만들어지는 경우 자동 검증 규칙에서 제외해야 한다.

```php
use Spatie\LaravelData\Attributes\WithoutValidation;

class UserData extends Data
{
    public function __construct(
        #[WithoutValidation] // name 은 요청에 없으므로 검증 규칙에서 제외
        public string $name,

        public string $firstName,
        public string $lastName,
    ) {
    }

    public static function fromRequest(array $payload): self
    {
        return new self(
            name: "{$payload['first_name']} {$payload['last_name']}",
            firstName: $payload['first_name'],
            lastName: $payload['last_name'],
        );
    }
}
```

클래스 전체 검증을 끄고 싶다면 `config/data.php`의 `validation_strategy`를 `Disabled`로 바꾸거나, 팩토리를 사용한다.

```php
// 이 호출 한정으로 검증을 끄고 데이터 객체만 생성
SongData::factory()->withoutValidation()->from($payload);
```

---

## 11. 예외 처리 - 검증 실패 시 동작

### 11-1. 기본 동작: `FormRequest`와 동일하다

laravel-data의 검증은 내부적으로 Laravel의 표준 `Illuminate\Validation\Validator`를 그대로 사용한다. 검증 실패 시 던져지는 예외도 우리에게 익숙한 **`Illuminate\Validation\ValidationException`** 이고, 기본 처리 방식도 `FormRequest`와 동일하다.

- **웹 요청(HTML 응답 기대)**: 검증 실패 시 이전 페이지로 리다이렉트 + 세션에 에러 메시지 저장(`$errors` 변수로 blade에서 사용 가능)
- **API 요청(JSON 응답 기대, `Accept: application/json`)**: `422 Unprocessable Entity` + 아래와 같은 JSON 바디 자동 반환

```json
{
    "message": "The title field is required. (and 1 more error)",
    "errors": {
        "title": ["The title field is required."],
        "artist": ["The artist field is required."]
    }
}
```

즉, 컨트롤러 쪽에서 별도로 try/catch 하지 않아도 기존에 `FormRequest`를 쓸 때와 동일한 사용자 경험을 얻는다. Laravel의 전역 예외 핸들러(`bootstrap/app.php`의 `withExceptions`)가 `ValidationException`을 표준 방식으로 렌더링해주기 때문이다.

### 11-2. 검증 실행 시점

- 컨트롤러 인자로 Data 클래스를 타입힌트 → 자동 검증
- `DataClass::from($request)`처럼 `Request`/`FormRequest`를 넘길 때 → 자동 검증
- `app(DataClass::class)`처럼 컨테이너에서 resolve → 자동 검증
- 순수 배열을 넘기는 `DataClass::from($array)` → 기본 전략(`OnlyRequests`)에서는 **검증 안 함**. 검증까지 원하면 `validateAndCreate()`를 명시적으로 호출해야 한다.

```php
// 배열 입력을 명시적으로 검증하고 싶을 때
try {
    $song = SongData::validateAndCreate($request->all());
} catch (\Illuminate\Validation\ValidationException $e) {
    // $e->errors() : ['title' => ['The title field is required.'], ...]
    // $e->validator : 원본 Validator 인스턴스
    return response()->json([
        'message' => '입력값을 확인해주세요.',
        'errors' => $e->errors(),
    ], 422);
}
```

### 11-3. 에러 메시지 / 속성명 / 리다이렉트 커스터마이징

`FormRequest`에서 `messages()`, `attributes()`를 오버라이드하던 것과 동일하게, Data 클래스에도 같은 이름의 메서드를 둘 수 있다.

```php
class SongData extends Data
{
    public function __construct(
        public string $title,
        public string $artist,
    ) {
    }

    // 검증 실패 메시지 커스터마이징 (FormRequest::messages() 와 동일)
    public static function messages(): array
    {
        return [
            'title.required' => '제목은 필수 입력 항목입니다.',
            'artist.required' => '아티스트는 필수 입력 항목입니다.',
        ];
    }

    // 에러 메시지에 노출될 필드명 커스터마이징 (FormRequest::attributes() 와 동일)
    public static function attributes(): array
    {
        return ['title' => '노래 제목'];
    }

    // 검증 실패 시 리다이렉트할 경로 지정 (웹 라우트에서 유용)
    public static function redirect(): string
    {
        return '/songs/create';
    }

    // 또는 이름 있는 라우트로 리다이렉트
    public static function redirectRoute(): string
    {
        return 'songs.create';
    }

    // 첫 번째 검증 실패에서 즉시 중단할지 여부
    public static function stopOnFirstFailure(): bool
    {
        return true;
    }

    // 여러 폼이 한 페이지에 있을 때 에러백 이름 분리
    public static function errorBag(): string
    {
        return 'songForm';
    }
}
```

이 메서드들은 모두 **의존성 주입**을 받을 수 있다 (예: 다국어 메시지를 리포지토리에서 가져오기).

### 11-4. `withValidator()` 훅으로 여러 필드 조합 검증

```php
use Illuminate\Validation\Validator;

class SongData extends Data
{
    public function __construct(
        public string $title,
        public ?string $releaseDate,
    ) {
    }

    public static function withValidator(Validator $validator): void
    {
        $validator->after(function (Validator $validator) {
            if ($validator->errors()->isNotEmpty()) {
                return; // 이미 실패했으면 추가 검증 생략
            }

            // 여러 필드를 조합한 커스텀 검증 로직
            if (str_contains($validator->getData()['title'] ?? '', '금지어')) {
                $validator->errors()->add('title', '제목에 금지어를 포함할 수 없습니다.');
            }
        });
    }
}
```

> `withValidator()`는 **루트 Data 객체에만 호출**된다. 중첩된 Data 객체나 컬렉션 내부의 Data 객체에서는 호출되지 않는다.

### 11-5. 전역 예외 핸들러에서 API 응답 포맷 통일하기

여러 Data 클래스에서 반복되는 JSON 에러 포맷을 통일하고 싶다면, `FormRequest`를 쓸 때와 마찬가지로 전역 예외 핸들러에서 `ValidationException` 렌더링을 한 번에 커스터마이징하면 된다 (laravel-data 전용 기능이 아니라 Laravel 표준 방식이 그대로 적용된다).

```php
// bootstrap/app.php (Laravel 11+)
use Illuminate\Validation\ValidationException;

->withExceptions(function ($exceptions) {
    $exceptions->render(function (ValidationException $e, $request) {
        if ($request->expectsJson()) {
            return response()->json([
                'success' => false,
                'message' => '입력값 검증에 실패했습니다.',
                'errors' => $e->errors(),
            ], 422);
        }
        // 웹 요청은 기본 리다이렉트 동작에 맡긴다.
    });
})
```

### 11-6. 요약표

| 상황 | 처리 방식 |
|---|---|
| 컨트롤러에서 Data 자동 주입 시 검증 실패 | 자동으로 `ValidationException` throw → Laravel 기본 핸들러가 처리 (별도 코드 불필요) |
| 배열을 직접 `validateAndCreate()`로 검증 | `try/catch`로 `ValidationException` 직접 처리 |
| 에러 메시지/필드명/리다이렉트 커스터마이징 | Data 클래스에 `messages()`, `attributes()`, `redirect()` 등 정의 |
| 여러 필드 조합 검증 | `withValidator()` 훅 사용 |
| API 에러 응답 포맷 전역 통일 | `bootstrap/app.php`의 `withExceptions()`에서 `ValidationException` 렌더링 오버라이드 |

---

## 12. FormRequest → Data 마이그레이션 비교

| 항목 | FormRequest | laravel-data `Data` |
|---|---|---|
| 검증 규칙 정의 | `rules()` 메서드에 배열로 작성 | 프로퍼티 타입 자동 추론 + Attribute + (필요 시) `rules()` |
| 인가(authorize) | `authorize()` 메서드 지원 | **미지원** — Policy/Gate나 미들웨어로 별도 처리 |
| 검증된 값 사용 | `$request->validated()` (배열) | 타입 안전한 객체 프로퍼티로 바로 접근 |
| 응답 변환(API Resource) | 별도 `JsonResource` 필요 | 같은 클래스가 `toArray()`/`toResource()`도 담당 |
| 검증 실패 시 예외 | `ValidationException` | 동일한 `ValidationException` (Laravel 표준) |
| 에러 메시지/속성명 커스터마이징 | `messages()`, `attributes()` | 동일한 이름의 static 메서드 |
| 중첩 데이터 검증 | 수동으로 `items.*.name` 같은 dot 표기 작성 | 중첩 Data 클래스만 선언하면 자동 생성 |
| TypeScript 타입 생성 | 없음 | `spatie/laravel-typescript-transformer` 연동 가능 |

**실전 팁**: 인가 로직이 복잡한 엔드포인트는 `FormRequest`를 완전히 버리기보다, **인가만 `FormRequest`나 Policy가 담당하고, 검증+DTO 변환은 `Data` 클래스가 담당**하도록 역할을 나누는 것도 좋은 선택이다. `Data::from($request)`에 `FormRequest` 인스턴스를 그대로 넘길 수 있으므로 두 패턴은 자연스럽게 공존 가능하다.

```php
class UpdateSongController
{
    // authorize() 는 SongRequest(FormRequest)가 담당, 검증 규칙/DTO 변환은 SongData가 담당
    public function __invoke(Song $model, SongRequest $request): RedirectResponse
    {
        $model->update(SongData::from($request)->all());

        return redirect()->back();
    }
}
```

---

## 13. Resource로 사용하기 (응답 변환)

Data 클래스는 응답을 만들 때도 그대로 쓸 수 있어, `JsonResource`를 따로 만들 필요가 없는 경우가 많다.

```php
class SongController
{
    public function show(Song $song): SongData
    {
        // 컨트롤러에서 Data 객체를 그대로 반환하면 Laravel이 자동으로 JSON으로 직렬화한다.
        return SongData::from($song);
    }

    public function index()
    {
        // toArray() 로 명시적으로 배열 변환도 가능
        return SongData::collect(Song::all())->toArray();
    }
}
```

민감하거나 무거운 관계 데이터는 `Lazy`로 감싸 필요할 때만 응답에 포함시킬 수 있다.

```php
use Spatie\LaravelData\Lazy;

class SubscriberData extends Data
{
    public function __construct(
        public readonly ?int $id,
        public readonly string $email,
        public readonly null|Lazy|DataCollection $tags, // include('tags') 를 호출해야만 포함됨
    ) {
    }
}

// 응답 시 필요한 것만 선택적으로 포함
$data->include('tags')->toJson();
```

---

## 14. 실무 체크리스트

- [ ] `app/Data/` 같은 전용 디렉토리에 Data 클래스를 모아 관리한다 (`make:data` 커맨드의 기본 네임스페이스이기도 하다).
- [ ] 프로퍼티는 가능하면 `readonly`로 선언해 불변성을 유지한다.
- [ ] 타입 추론으로 커버되지 않는 제약(형식, 범위, DB 제약 등)만 Validation Attribute로 명시한다 — `required`, `string` 같은 건 직접 쓰지 않는다.
- [ ] 인가(authorize) 로직은 Data 클래스가 아니라 Policy/Gate 또는 미들웨어에 둔다.
- [ ] API/웹 양쪽에서 쓰이는 프로젝트라면 전역 예외 핸들러에서 `ValidationException` 응답 포맷을 한 번만 통일해둔다.
- [ ] 배열을 직접 검증해야 하는 서비스/커맨드 계층 코드에서는 `from()` 대신 `validateAndCreate()`를 명시적으로 사용한다 (기본 전략은 요청에서만 검증하기 때문).
- [ ] DDD 관점에서는 Data 클래스가 여전히 **애플리케이션/HTTP 경계의 형식 검증**만 책임지도록 하고, 도메인 불변식 검증은 도메인 계층에 둔다.

# NativePHP for Mobile 실무 가이드 (v4 기준)

> 기준일: 2026년 8월 / 문서 출처: https://nativephp.com/docs/mobile/4
> Laravel 개발 경험이 있다는 전제로 작성했습니다.

---

## 0. 큰 그림부터

NativePHP for Mobile은 **Laravel 앱을 iOS/Android 네이티브 앱으로 만들어 주는 Composer 패키지**입니다.

핵심 개념 세 가지:

1. **웹서버가 없습니다.** 앱 안에 PHP 런타임이 통째로 내장(embed)됩니다. 서버에 요청을 보내는 게 아니라, 기기 안에서 PHP가 직접 돕니다. 완전한 오프라인 동작이 기본값입니다.
2. **v4부터는 웹뷰가 기본이 아닙니다.** v3까지는 "네이티브 껍데기 + 웹뷰에 Blade 렌더링" 구조였지만, v4의 **SuperNative** 아키텍처는 Blade 문법으로 **진짜 네이티브 UI 위젯**(SwiftUI / Jetpack Compose)을 그립니다. `<native:button>`은 스타일링된 `<div>`가 아니라 그 기기의 실제 네이티브 버튼입니다.
3. **웹뷰도 여전히 쓸 수 있습니다.** 기본값이 아니라 "컴포넌트 중 하나"로 강등됐을 뿐입니다. 기존 웹뷰 방식 앱을 그대로 유지하면서 화면 단위로 하나씩 네이티브로 옮겨갈 수 있습니다.

### Laravel 개발자 입장에서의 대응 관계

| Laravel 웹 | NativePHP Mobile v4 |
|---|---|
| `routes/web.php` + `Route::get()` | `routes/mobile.php` + `Route::native()` |
| Livewire 컴포넌트 | `NativeComponent` (거의 동일한 감각) |
| Blade 뷰 + HTML 태그 | Blade 뷰 + `<native:*>` EDGE 컴포넌트 |
| `redirect()->route()` | `$this->navigate('/path')` |
| 브라우저 뒤로가기 | 네이티브 내비게이션 스택 (`$this->back()`) |
| `php artisan serve` | `php artisan native:run` |

---

## 1. 개발 환경 준비

### 1.1 공통 요구사항

- PHP 8.3 이상
- Laravel 11 이상
- (Mac/Windows 기준) PHP 설치가 아직이라면 Laravel Herd가 가장 간편합니다.

### 1.2 중요한 제약 사항 — 먼저 확인하세요

| 제약 | 내용 |
|---|---|
| **WSL 미지원** | NativePHP는 WSL(Windows Subsystem for Linux)에서 동작하지 않습니다. Windows에 직접 설치해서 실행해야 합니다. |
| **iOS는 Mac 필수** | iOS 앱 컴파일은 Mac에서만 가능합니다(Apple 제약). Apple Silicon(M1 이상) + macOS 15.6 이상이 필요합니다. |
| **Windows Defender** | `C:\temp`와 프로젝트 폴더를 예외 목록에 추가하세요. 빌드 중 생성되는 임시 파일을 실시간 검사하면서 속도가 크게 느려집니다. |

> **WSL을 주력으로 쓰신다면**: macOS 환경에서 작업하시는 게 훨씬 매끄럽습니다. iOS까지 목표라면 사실상 Mac이 필수이므로, macOS를 메인으로 잡으시는 걸 권합니다.
> Mac이 없는데 iOS 빌드가 필요하다면 공식 클라우드 빌드 서비스 **Bifrost**(월 $10~)를 대안으로 검토해 볼 수 있습니다.

### 1.3 iOS 환경 구축

```bash
# 1. Xcode 26 이상 설치 (Mac App Store)

# 2. Command Line Tools
xcode-select --install
xcode-select -p   # 설치 확인

# 3. Homebrew (없다면)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 4. CocoaPods
brew install cocoapods
pod --version     # 설치 확인
```

**Apple Developer Program($99/년) 필요 여부**
- 시뮬레이터에서 개발·테스트만 → **불필요**
- 실기기 테스트 / App Store 배포 / 푸시 알림 테스트 → **필요**

### 1.4 Android 환경 구축

1. Android Studio 2024.2.1 이상 설치
2. **Tools → SDK Manager → SDK Platforms**에서 API 29 이상 플랫폼을 최소 1개 설치 (최신 안정판은 Android 16 / API 36)
3. **SDK Tools** 탭에서 `Android SDK Build-Tools`, `Android SDK Platform-Tools` 설치 확인
4. Windows 사용자는 **7zip**도 설치해야 합니다.

**JDK 주의점**: 최근 Android Studio는 JDK를 자동 설치하지 않습니다. Gradle 오류가 나면 Gradle-JDK 호환성 매트릭스를 확인하세요. 최신 JDK가 아직 지원되지 않는 경우가 흔합니다. 설치된 Gradle 버전은 `native:install` 실행 후 `nativephp/android/.gradle` 폴더에서 확인할 수 있습니다.

**환경변수 설정 (macOS)**

```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 17)
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$JAVA_HOME/bin:$ANDROID_HOME/emulator:$ANDROID_HOME/tools:$ANDROID_HOME/tools/bin:$ANDROID_HOME/platform-tools
```

**환경변수 설정 (Windows)**

```
set ANDROID_HOME=C:\Users\yourname\AppData\Local\Android\Sdk
set JAVA_HOME=C:\Program Files\Microsoft\jdk-17.0.8.7-hotspot
set PATH=%PATH%;%JAVA_HOME%\bin;%ANDROID_HOME%\platform-tools
```

**검증**: 터미널에서 `java -version`, `adb devices` 두 명령이 정상 동작하면 준비 완료입니다.

> "No AVDs found" 오류가 나면 Android Studio의 Virtual Devices에서 가상 기기를 최소 1개 만들어 주세요.

---

## 2. 설치

### 2.1 순서

```bash
# 1) 새 Laravel 앱 생성 (권장)
laravel new my-mobile-app
cd my-mobile-app

# 2) 패키지 설치
composer require nativephp/mobile
```

### 2.2 .env 설정 — `native:install` **전에** 반드시

```dotenv
NATIVEPHP_APP_ID=com.yourcompany.yourapp
NATIVEPHP_DEVELOPMENT_TEAM={애플 팀 ID}   # iOS 실기기/배포 시
```

- `NATIVEPHP_APP_ID`는 소문자·숫자·마침표만 사용하세요. 하이픈, 언더스코어, 공백, 이모지가 들어가면 빌드가 실패합니다.
- 팀 ID는 Apple Developer 계정 → Membership details에서 확인합니다.

### 2.3 설치 실행

```bash
php artisan native:install
php artisan native:run
```

**ICU 바이너리 선택 프롬프트**가 뜹니다.
- `intl` 확장에 의존하는 앱 → ICU 포함 버전 선택
- **Filament를 쓸 계획이면 ICU 필수**
- 잘 모르겠고 특별히 필요 없다면 → 기본값(비-ICU) 선택

### 2.4 `nativephp` 디렉터리

설치 후 프로젝트 루트에 `nativephp/` 디렉터리와 `config/nativephp.php`가 생깁니다.

- `nativephp/`에는 iOS/Android 네이티브 프로젝트 파일이 들어 있습니다.
- **직접 열거나 수정할 일은 거의 없습니다.**
- **이 디렉터리는 일회성(ephemeral)으로 취급하세요.** 패키지 업그레이드 시 `php artisan native:install --force`를 실행하면 이 디렉터리를 통째로 지우고 다시 만듭니다.
- 따라서 **`.gitignore`에 `nativephp/`를 추가**하는 것이 권장 사항입니다.

### 2.5 실기기에서 실행하기

| 플랫폼 | 조건 |
|---|---|
| iOS | 기기를 Developer Mode로 전환 + Apple Developer 계정에 등록된 기기여야 함 |
| Android | 개발자 옵션 활성화 + USB 디버깅(ADB) 활성화 |

시뮬레이터/에뮬레이터만으로도 개발과 테스트가 가능하지만, 스토어 제출 전에는 반드시 실기기 검증을 거치는 게 안전합니다.

> **팁**: 네이티브로 띄우기 전에 먼저 브라우저에서 `php artisan serve`로 앱을 실행해 보세요. 네이티브 컨텍스트에서는 잡기 어려운 예외를 미리 걸러낼 수 있습니다.

---

## 3. SuperNative 아키텍처 이해하기

v4의 핵심입니다. 세 가지 아이디어가 맞물려 돌아갑니다.

### 3.1 PHP와 네이티브 레이어의 공유 메모리

네이티브 레이어와 PHP 애플리케이션이 **메모리를 직접 공유**합니다. 네트워크 왕복도, 직렬화 오버헤드도, 웹뷰 브리지 대기도 없습니다. 상태 변경이 PHP와 네이티브 UI 사이를 거의 즉시 오갑니다.

Laravel 관점에서 비유하자면, HTTP 요청/응답 사이클을 거치는 Livewire와 달리 **같은 프로세스 메모리 안에서 컴포넌트 상태와 뷰가 붙어 있는** 구조입니다.

### 3.2 Livewire 유사 컴포넌트

각 화면이 `NativeComponent` 하나로 구동됩니다. 프로퍼티가 상태이고, 메서드가 액션이며, `render()`가 Blade 뷰를 반환합니다. Livewire를 써보셨다면 학습 곡선이 거의 없습니다.

### 3.3 EDGE 컴포넌트

`<native:button>`, `<native:list>` 같은 태그가 플랫폼 고유 UI 프레임워크의 실제 위젯으로 매핑됩니다. 스크롤 물리, 텍스트 렌더링, 전환 애니메이션, 컨텍스트 메뉴, 다크 모드, 다이내믹 타입, 스크린 리더 — 브라우저에서 흉내 내야 했던 것들이 플랫폼 네이티브에서는 기본으로 제공됩니다.

### 3.4 웹뷰 방식 그대로 유지하기 (마이그레이션 전략)

v3 이하 방식 앱을 그대로 살리려면:

```php
// routes/mobile.php
Route::native('/home', WebViewScreen::class);
```

```blade
{{-- resources/views/webviewscreen.blade.php --}}
<webview php url="/" fullscreen />
```

```php
// routes/web.php
Route::view('/', 'welcome');
```

```dotenv
NATIVEPHP_START_URL=/home
```

이렇게 두면 기존 웹뷰 기반 앱이 그대로 동작하고, 준비될 때 화면 단위로 하나씩 SuperNative로 전환할 수 있습니다. 전환하지 않아도 무방합니다.

---

## 4. 라우팅과 화면 이동

### 4.1 라우트 등록

네이티브 화면은 iOS `UINavigationController` / Android Activity 스택처럼 **스택 구조**를 이룹니다.

```php
// routes/mobile.php
use App\NativeComponents\Home;
use App\NativeComponents\ItemDetail;

Route::native('/', Home::class);
Route::native('/item/{id}', ItemDetail::class);
```

Livewire의 `Route::livewire()` 자리에 `Route::native()` 매크로를 쓴다고 보시면 됩니다. 라우트 파라미터는 Laravel 웹 라우트와 동일하게 동작합니다.

### 4.2 화면 이동 API

```php
// 새 화면 push (기본: 오른쪽에서 슬라이드 인)
$this->navigate('/item/42');

// 데이터 전달
$this->navigate('/item/42', ['source' => 'home-feed']);

// 뒤로 가기 (스택에서 pop)
$this->back();

// 현재 화면 교체 (로그인/로그아웃 후에 유용, 기본: 페이드)
$this->replace('/login');

// 네이티브 스택을 걷어내고 웹뷰로 진입
$this->exitToWeb('/dashboard');
```

### 4.3 파라미터와 데이터 읽기

```php
class ItemDetail extends NativeComponent
{
    public function mount(): void
    {
        $id     = $this->param('id');                  // 라우트 URI에서
        $source = $this->data('source', 'unknown');    // navigate() 두 번째 인자에서
    }
}
```

두 접근자 모두 키가 없을 때 반환할 기본값을 두 번째 인자로 받습니다.

### 4.4 이름 있는 라우트

```php
$this->navigate($this->route('listing.show', ['id' => 5]));
```

Laravel의 URL 제너레이터에 `absolute: false`로 위임합니다.

### 4.5 전환 애니메이션

```php
use Native\Mobile\Edge\Transition;

$this->navigate('/item/42')->transition(Transition::SlideFromBottom);
```

사용 가능한 값:

| 값 | 설명 |
|---|---|
| `SlideFromRight` | `navigate()` 기본값 |
| `SlideFromLeft` | `back()` 기본값 |
| `SlideFromBottom` | 모달 스타일 표현 |
| `Fade` | `replace()` 기본값 |
| `FadeFromBottom` | 은은한 수직 페이드 |
| `ScaleFromCenter` | 줌인 효과 |
| `ParallaxPush` | 들어오는 화면은 우측에서, 나가는 화면은 좌측으로 밀리는 패럴랙스 |
| `None` | 애니메이션 없음 |

### 4.6 Blade에서 직접 내비게이션

컴포넌트 메서드를 만들 필요 없이 `@navigate` 디렉티브로 처리할 수 있습니다. 어떤 노드에든 붙일 수 있어 모든 요소가 내비게이션 타깃이 됩니다.

```blade
{{-- 기본형 --}}
<native:pressable @navigate="/item/42" class="w-full p-4">
    <native:text>항목 보기</native:text>
</native:pressable>

{{-- 데이터 전달 --}}
<native:pressable @navigate="'/item/42', ['source' => 'home-feed']">
    <native:text>항목 보기</native:text>
</native:pressable>

{{-- 전환 지정 --}}
<native:pressable @navigate.slideFromBottom="/item/42">
    <native:text>모달로 열기</native:text>
</native:pressable>

{{-- 액션 지정 --}}
<native:pressable @navigate.back>
    <native:text>뒤로</native:text>
</native:pressable>

<native:pressable @navigate.replace="/login">
    <native:text>로그아웃</native:text>
</native:pressable>

<native:pressable @navigate.exitToWeb="/dashboard">
    <native:text>대시보드 열기</native:text>
</native:pressable>
```

전환 modifier와 액션 modifier는 조합할 수 있습니다 — `@navigate.replace.fade="/login"`

전환 modifier 목록: `fade`, `slideFromRight`, `slideFromLeft`, `slideFromBottom`, `fadeFromBottom`, `scaleFromCenter`, `parallaxPush`, `none`

### 4.7 Android 하드웨어 뒤로가기 제어

기본적으로 기기 뒤로가기 버튼은 스택을 pop합니다. `onBackPressed()`를 오버라이드해 커스텀 동작을 넣을 수 있습니다.

```php
class CheckoutForm extends NativeComponent
{
    public bool $dirty = false;

    public function onBackPressed(): void
    {
        if ($this->dirty) {
            $this->showDiscardConfirmation();
            return;
        }

        $this->back();
    }
}
```

오버라이드해도 최종적으로 pop해야 한다면 `$this->back()`을 직접 호출해 주세요.

> `<native:bottom-nav-item>` 탭은 자동으로 `replace` 시맨틱을 씁니다. 그래서 뒤로가기가 탭 히스토리를 하나씩 되짚지 않고 탭 섹션 전체를 한 번에 빠져나옵니다.

---

## 5. EDGE 컴포넌트 목록

Blade에서 `<native:*>` 형태로 사용하는 네이티브 UI 컴포넌트입니다. 실제 화면을 짤 때 참고용 목록으로 활용하세요.

**레이아웃 / 구조**
`column` · `row` · `stack` · `spacer` · `divider` · `scroll-view` · `layout(Layout & Styling)` · `safe-area` · `positioning`

**입력**
`button` · `button-group` · `text-input` · `checkbox` · `toggle` · `radio-group` · `select` · `slider` · `pressable` · `gesture-area`

**표시**
`text` · `icon` · `image` · `badge` · `chip` · `progress-bar` · `activity-indicator` · `shapes` · `canvas`

**목록 / 컬렉션**
`list` · `virtual-list` · `lazy-grid` · `carousel` · `accordion` · `refreshable`

**내비게이션 / 오버레이**
`top-bar` · `bottom-nav` · `side-nav` · `tab-row` · `modal` · `bottom-sheet` · `menus` · `fab(Floating Action Button)`

**기타**
`web-view` (웹뷰를 컴포넌트로 삽입)

---

## 6. 네이티브 기능 (Core Plugins)

네이티브 API는 플러그인으로 제공됩니다. 기본 제공되는 코어 플러그인은 다음과 같습니다.

| 플러그인 | 용도 |
|---|---|
| **Biometrics** | 지문/Face ID 인증 |
| **Browser** | 외부 브라우저 열기 |
| **Camera** | 사진 촬영 / 갤러리 |
| **Firebase** | 푸시 알림 |
| **Geolocation** | 위치 정보 |
| **Microphone** | 녹음 |
| **Network** | 네트워크 상태 감지 |
| **Scanner** | QR / 바코드 스캔 |
| **SecureStorage** | 키체인·키스토어 기반 보안 저장소 |
| **Share** | 시스템 공유 시트 |
| **Vibe** | 진동 / 햅틱 |

### 6.1 사용 예 — 파일 이동

```php
use Native\Mobile\Facades\File;

$temp      = sys_get_temp_dir().'/recording.m4a';
$permanent = storage_path('recordings/recording.m4a');

if (File::move($temp, $permanent)) {
    // 저장 완료
}
```

파사드 형태라서 Laravel에서 `Storage::`, `Cache::` 쓰던 감각 그대로입니다. 이 함수들은 웹뷰 안의 JavaScript에서도 Native 라이브러리를 통해 호출할 수 있습니다.

### 6.2 플러그인 설치

플러그인은 일반 Composer 패키지입니다.

```bash
composer require {vendor}/{plugin}
```

**중요**: PHP 서비스 프로바이더는 Laravel이 자동 발견하지만, **네이티브 코드는 명시적으로 등록해야 빌드에 포함됩니다.** 이는 전이 의존성(transitive dependency)이 개발자 동의 없이 네이티브 코드를 끌고 들어오는 것을 막기 위한 보안 조치입니다.

```bash
# NativeServiceProvider 퍼블리시 (아직 안 했다면)
php artisan native:install --publish

# 플러그인 등록 → app/Providers/NativeServiceProvider.php에 추가됨
# 등록 후 재빌드 필요
php artisan native:run
```

유료 플러그인은 별도의 프라이빗 Composer 저장소로 배포되며, 구매 대시보드에서 인증 정보를 받아 Composer에 설정해야 합니다.

---

## 7. 심화 주제 (Digging Deeper)

공식 문서에 별도 섹션으로 정리되어 있는 항목들입니다. 실제 앱을 만들 때 순서대로 참고하시면 됩니다.

| 주제 | 왜 중요한가 |
|---|---|
| **Lifecycle Hooks** | 앱 시작/일시정지/재개 시점 처리 |
| **Data Binding** | 컴포넌트 프로퍼티 ↔ UI 양방향 바인딩 |
| **Reactivity** | 상태 변경 시 UI 갱신 메커니즘 |
| **Theming** | 다크 모드, 플랫폼별 디자인 토큰 |
| **Gestures & Animation** | 스와이프, 드래그, 전환 효과 |
| **Databases** | 기기 내 SQLite — Eloquent 그대로 사용 |
| **Queues** | 기기 내 큐 워커 |
| **Authentication** | 로컬 인증 + 서버 연동 |
| **Security** | 번들에 포함되는 코드/자산 보호 |
| **Deep Links** | 외부에서 앱 특정 화면 진입 |
| **WebSockets** | 실시간 통신 |
| **Push Notifications** | Firebase 기반 |

---

## 8. 테스트

전용 테스트 API가 v4에 포함되어 있습니다.

- **Interactions** — 탭, 입력 등 사용자 상호작용 시뮬레이션
- **Native Events & the Bridge** — 네이티브 이벤트/브리지 호출 검증
- **Navigation & Flows** — 화면 전환 플로우 테스트
- **Accessibility** — 접근성 검증
- **Advanced** — 고급 시나리오

Pest/PHPUnit 기반이라 기존 Laravel 테스트 작성 방식이 그대로 이어집니다.

---

## 9. 배포

`Publishing Your App` 섹션에 플랫폼별 가이드가 있습니다.

### 9.1 Android
- Google Play Console 등록
- 서명 키(keystore) 생성 및 설정
- AAB 빌드 및 업로드

### 9.2 iOS
- Apple Developer Program 가입 필수 ($99/년)
- 프로비저닝 프로파일 / 인증서 설정
- App Store Connect 업로드

### 9.3 버전 관리 env

```dotenv
NATIVEPHP_APP_VERSION="1.0.0"
NATIVEPHP_APP_VERSION_CODE="1"
```

---

## 10. 실무 진행 순서 제안

처음 시작하신다면 이 순서를 권합니다.

1. **환경부터 완전히 검증** — `java -version`, `adb devices`, `pod --version`이 모두 통과하는지 확인. 여기서 막히는 시간이 의외로 깁니다.
2. **Kitchen Sink 데모 먼저 실행** — 공식 데모 앱을 돌려서 환경이 정상인지 확인하고, 어떤 컴포넌트가 있는지 눈으로 봅니다.
3. **SuperNative 데모 클론**
   ```bash
   git clone https://github.com/nativephp/super-native
   cd super-native
   composer install
   php artisan native:install
   php artisan native:run
   ```
   소스를 읽으면서 화면 구성 방식을 익힌 뒤, 하나씩 자기 화면으로 바꿔 나가는 방식이 가장 빠릅니다.
4. **작은 앱 하나를 끝까지** — 화면 3~4개짜리로 라우팅 · 상태 · 네이티브 API 하나(카메라나 위치) · 빌드까지 한 사이클을 완주해 보세요.
5. **그다음 실제 프로젝트로** — 이 시점에 데이터베이스, 인증, 푸시 등 심화 주제를 붙입니다.

---

## 11. 참고 링크

| 항목 | URL |
|---|---|
| Mobile v4 공식 문서 | https://nativephp.com/docs/mobile/4/getting-started/introduction |
| Quick Start | https://nativephp.com/docs/mobile/4/getting-started/quick-start |
| Environment Setup | https://nativephp.com/docs/mobile/4/getting-started/environment-setup |
| SuperNative 소개 | https://nativephp.com/docs/mobile/4/architecture/super-native |
| EDGE 컴포넌트 목록 | https://nativephp.com/docs/mobile/4/edge-components/introduction |
| 코어 플러그인 | https://nativephp.com/docs/mobile/4/plugins/introduction |
| GitHub (mobile) | https://github.com/nativephp/mobile-air |
| SuperNative 데모 | https://github.com/nativephp/super-native |
| Packagist | https://packagist.org/packages/nativephp/mobile |
| Discord 커뮤니티 | https://discord.gg/nativephp |
| Bifrost (클라우드 빌드) | https://bifrost.nativephp.com |

---

## 12. 짚고 넘어갈 점

- **라이선스**: 초기 버전(v1~)에서는 유료 라이선스 구매가 필수였으나, 현재 Packagist에는 MIT 라이선스로 표기되어 있습니다. 상용 프로젝트에 적용하실 계획이라면 최신 라이선스 조건을 공식 채널에서 한 번 더 확인하시는 게 안전합니다.
- **PHP 버전 고정**: 모바일 번들 PHP는 특정 버전(최근 기준 8.4대)으로 고정됩니다. 애플리케이션 코드가 그 버전에서 동작하는지 확인이 필요합니다.
- **성숙도**: v1에서 v4까지 1년 남짓한 사이에 아키텍처가 크게 바뀌었습니다. 빠르게 발전하는 만큼 breaking change 가능성도 있으니, 프로덕션 도입 전에는 Upgrade Guide와 Support Policy를 확인하시는 걸 권합니다.

---
title: NativePHP Core Plugins
category: frontend
tags: [frontend, nativephp, mobile, plugins]
related: [[NativePHP Mobile Installation]], [[NativePHP Mobile Overview]]
---

# NativePHP Core Plugins

카메라, 위치, 생체 인증 같은 기기 네이티브 API는 플러그인으로 제공된다.

## 핵심 개념

### 기본 제공 코어 플러그인

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

### 사용 예 — 파일 이동

```php
use Native\Mobile\Facades\File;

$temp      = sys_get_temp_dir().'/recording.m4a';
$permanent = storage_path('recordings/recording.m4a');

if (File::move($temp, $permanent)) {
    // 저장 완료
}
```

파사드 형태라서 Laravel에서 `Storage::`, `Cache::` 쓰던 감각 그대로다. 이 함수들은 웹뷰 안의 JavaScript에서도 Native 라이브러리를 통해 호출할 수 있다.

## Laravel 구현

### 플러그인 설치

플러그인은 일반 Composer 패키지다.

```bash
composer require {vendor}/{plugin}
```

**중요**: PHP 서비스 프로바이더는 Laravel이 자동 발견하지만, **네이티브 코드는 명시적으로 등록해야 빌드에 포함된다.** 이는 전이 의존성(transitive dependency)이 개발자 동의 없이 네이티브 코드를 끌고 들어오는 것을 막기 위한 보안 조치다.

```bash
# NativeServiceProvider 퍼블리시 (아직 안 했다면)
php artisan native:install --publish

# 플러그인 등록 → app/Providers/NativeServiceProvider.php에 추가됨
# 등록 후 재빌드 필요
php artisan native:run
```

유료 플러그인은 별도의 프라이빗 Composer 저장소로 배포되며, 구매 대시보드에서 인증 정보를 받아 Composer에 설정해야 한다.

## 주의사항 / 안티패턴

- `composer require`만 하고 `NativeServiceProvider`에 네이티브 코드 등록을 빠뜨리면, PHP 코드는 정상 컴파일되는데 실제 기기에서 플러그인 기능이 동작하지 않는 상태가 된다 — Laravel의 자동 발견에 익숙하면 놓치기 쉬운 단계다.
- 플러그인을 새로 등록한 뒤에는 반드시 재빌드(`native:run`)해야 반영된다 — 핫 리로드로는 네이티브 코드 변경이 적용되지 않는다.
- Biometrics/Camera/Microphone/Geolocation처럼 사용자 권한이 필요한 플러그인은 플랫폼별 권한 요청 UX(iOS Info.plist 설명 문구, Android 런타임 권한)를 별도로 챙겨야 스토어 심사에서 반려되지 않는다.

## 참고

- [[NativePHP Mobile Installation]] — `native:install --publish`와 초기 설치 흐름
- [[NativePHP Mobile Overview]] — 전체 개념
- 소스: `2026-08-12_NativePHP for Mobile 실무 가이드.md`

---
title: NativePHP Mobile Installation
category: frontend
tags: [frontend, nativephp, mobile, setup]
related: [[NativePHP Mobile Environment Setup]], [[NativePHP Mobile Overview]], [[NativePHP Core Plugins]]
---

# NativePHP Mobile Installation

Laravel 앱에 NativePHP for Mobile을 설치하고 처음 실행하는 절차.

## 핵심 개념

### 설치 순서

```bash
# 1) 새 Laravel 앱 생성 (권장)
laravel new my-mobile-app
cd my-mobile-app

# 2) 패키지 설치
composer require nativephp/mobile
```

### `.env` 설정 — `native:install` **전에** 반드시

```dotenv
NATIVEPHP_APP_ID=com.yourcompany.yourapp
NATIVEPHP_DEVELOPMENT_TEAM={애플 팀 ID}   # iOS 실기기/배포 시
```

- `NATIVEPHP_APP_ID`는 소문자·숫자·마침표만 사용한다. 하이픈, 언더스코어, 공백, 이모지가 들어가면 빌드가 실패한다.
- 팀 ID는 Apple Developer 계정 → Membership details에서 확인한다.

### 설치 실행

```bash
php artisan native:install
php artisan native:run
```

**ICU 바이너리 선택 프롬프트**가 뜬다.
- `intl` 확장에 의존하는 앱 → ICU 포함 버전 선택
- **Filament를 쓸 계획이면 ICU 필수**
- 잘 모르겠고 특별히 필요 없다면 → 기본값(비-ICU) 선택

## Laravel 구현

### `nativephp` 디렉터리

설치 후 프로젝트 루트에 `nativephp/` 디렉터리와 `config/nativephp.php`가 생긴다.

- `nativephp/`에는 iOS/Android 네이티브 프로젝트 파일이 들어 있다.
- **직접 열거나 수정할 일은 거의 없다.**
- **이 디렉터리는 일회성(ephemeral)으로 취급한다.** 패키지 업그레이드 시 `php artisan native:install --force`를 실행하면 이 디렉터리를 통째로 지우고 다시 만든다.
- 따라서 **`.gitignore`에 `nativephp/`를 추가**하는 것이 권장 사항이다.

### 실기기에서 실행하기

| 플랫폼 | 조건 |
|---|---|
| iOS | 기기를 Developer Mode로 전환 + Apple Developer 계정에 등록된 기기여야 함 |
| Android | 개발자 옵션 활성화 + USB 디버깅(ADB) 활성화 |

시뮬레이터/에뮬레이터만으로도 개발과 테스트가 가능하지만, 스토어 제출 전에는 반드시 실기기 검증을 거치는 게 안전하다.

> **팁**: 네이티브로 띄우기 전에 먼저 브라우저에서 `php artisan serve`로 앱을 실행해 보라. 네이티브 컨텍스트에서는 잡기 어려운 예외를 미리 걸러낼 수 있다.

## 주의사항 / 안티패턴

- `NATIVEPHP_APP_ID`를 `native:install` **이후에** 바꾸면 반영이 꼬일 수 있다 — 반드시 설치 전에 최종 값을 정한다.
- `nativephp/` 디렉터리를 git에 커밋하고 수동으로 수정하면, 다음 `--force` 재생성 시 그 수정이 통째로 날아간다.
- 플러그인을 추가로 설치했다면 [[NativePHP Core Plugins]]의 네이티브 코드 등록 절차(`native:install --publish` + `NativeServiceProvider` 등록)를 빠뜨리지 않는다.

## 참고

- [[NativePHP Mobile Environment Setup]] — 설치 전 환경 준비
- [[NativePHP Mobile Overview]] — 전체 개념
- [[NativePHP Core Plugins]] — 설치 후 네이티브 기능 추가
- 소스: `2026-08-12_NativePHP for Mobile 실무 가이드.md`

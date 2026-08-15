---
title: NativePHP Mobile Environment Setup
category: frontend
tags: [frontend, nativephp, mobile, setup]
related: [[NativePHP Mobile Overview]], [[NativePHP Mobile Installation]]
---

# NativePHP Mobile Environment Setup

NativePHP for Mobile 개발을 시작하기 전 확인해야 할 요구사항과 제약, iOS/Android 환경 구축.

## 핵심 개념

### 공통 요구사항

- PHP 8.3 이상
- Laravel 11 이상
- (Mac/Windows 기준) PHP 설치가 아직이라면 Laravel Herd가 가장 간편

### 중요한 제약 사항 — 먼저 확인할 것

| 제약 | 내용 |
|---|---|
| **WSL 미지원** | NativePHP는 WSL(Windows Subsystem for Linux)에서 동작하지 않는다. Windows에 직접 설치해서 실행해야 한다. |
| **iOS는 Mac 필수** | iOS 앱 컴파일은 Mac에서만 가능하다(Apple 제약). Apple Silicon(M1 이상) + macOS 15.6 이상 필요. |
| **Windows Defender** | `C:\temp`와 프로젝트 폴더를 예외 목록에 추가한다. 빌드 중 생성되는 임시 파일을 실시간 검사하면서 속도가 크게 느려진다. |

> WSL을 주력으로 쓴다면 macOS 환경에서 작업하는 게 훨씬 매끄럽다. iOS까지 목표라면 사실상 Mac이 필수이므로, macOS를 메인으로 잡는 걸 권한다. Mac이 없는데 iOS 빌드가 필요하다면 공식 클라우드 빌드 서비스 **Bifrost**(월 $10~)를 대안으로 검토할 수 있다.

## Laravel 구현

### iOS 환경 구축

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
- 실기기 테스트 / App Store 배포 / 푸시 알림 테스트 → **필요** ([[NativePHP Mobile Deployment]] 참고)

### Android 환경 구축

1. Android Studio 2024.2.1 이상 설치
2. **Tools → SDK Manager → SDK Platforms**에서 API 29 이상 플랫폼을 최소 1개 설치 (최신 안정판은 Android 16 / API 36)
3. **SDK Tools** 탭에서 `Android SDK Build-Tools`, `Android SDK Platform-Tools` 설치 확인
4. Windows 사용자는 **7zip**도 설치해야 함

**JDK 주의점**: 최근 Android Studio는 JDK를 자동 설치하지 않는다. Gradle 오류가 나면 Gradle-JDK 호환성 매트릭스를 확인한다. 최신 JDK가 아직 지원되지 않는 경우가 흔하다. 설치된 Gradle 버전은 `native:install` 실행 후 `nativephp/android/.gradle` 폴더에서 확인할 수 있다.

```bash
# 환경변수 설정 (macOS)
export JAVA_HOME=$(/usr/libexec/java_home -v 17)
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$JAVA_HOME/bin:$ANDROID_HOME/emulator:$ANDROID_HOME/tools:$ANDROID_HOME/tools/bin:$ANDROID_HOME/platform-tools
```

```
:: 환경변수 설정 (Windows)
set ANDROID_HOME=C:\Users\yourname\AppData\Local\Android\Sdk
set JAVA_HOME=C:\Program Files\Microsoft\jdk-17.0.8.7-hotspot
set PATH=%PATH%;%JAVA_HOME%\bin;%ANDROID_HOME%\platform-tools
```

**검증**: 터미널에서 `java -version`, `adb devices` 두 명령이 정상 동작하면 준비 완료다.

> "No AVDs found" 오류가 나면 Android Studio의 Virtual Devices에서 가상 기기를 최소 1개 만든다.

## 주의사항 / 안티패턴

- WSL에서 개발 중이라면 이 단계에서 막힌다 — NativePHP 작업만큼은 Windows 네이티브 터미널이나 macOS로 옮겨야 한다.
- Windows Defender 예외 설정을 빠뜨리면 빌드 자체는 되지만 체감 속도가 크게 느려져서 "왜 이렇게 느리지"로 시간을 허비하기 쉽다.
- JDK 버전과 Gradle 요구 버전이 안 맞으면 에러 메시지가 JDK 문제라고 명확히 알려주지 않는 경우가 있다 — Gradle-JDK 호환성 매트릭스를 먼저 의심한다.

## 참고

- [[NativePHP Mobile Overview]] — 전체 개념
- [[NativePHP Mobile Installation]] — 환경 구축 이후 설치 단계
- [[NativePHP Mobile Deployment]] — Apple Developer Program이 필요해지는 배포 단계
- 소스: `2026-08-12_NativePHP for Mobile 실무 가이드.md`

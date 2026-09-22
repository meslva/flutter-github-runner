# Flutter: 한 번에 APK, AAB, IPA 만들기

Flutter 앱은 코드 하나로 Android와 iOS를 모두 만들 수 있죠. 러너에서도 마찬가지예요. 워크플로 하나로 **APK, AAB, IPA를 한 번에** 만들고, 테스트 배포와 스토어 업로드까지 할 수 있어요.

## 먼저 알아 두면 좋은 것

Flutter는 빌드할 때 결국 **Android는 Gradle, iOS는 Xcode**를 그대로 써요. 그래서:

- **서명과 업로드 방법은 네이티브 앱과 똑같아요.** 이 문서는 Flutter에서 달라지는 부분만 다루고, 자세한 설명은 [android.md](android.md)와 [ios.md](ios.md)로 연결해요.
- **달라지는 건 빌드 명령어, 결과물 위치, 버전 번호를 정하는 방법** 정도예요.

| 만들 것 | 명령어 | 결과물 위치 |
|---|---|---|
| 테스트용 APK | `flutter build apk --release` | `build/app/outputs/flutter-apk/app-release.apk` |
| 스토어용 AAB | `flutter build appbundle --release` | `build/app/outputs/bundle/release/app-release.aab` |
| IPA | `flutter build ipa --export-options-plist=...` | `build/ios/ipa/*.ipa` (archive는 `build/ios/archive/Runner.xcarchive`) |

## 목차
1. [러너 Mac 준비하기](#1-러너-mac-준비하기)
2. [프로젝트 설정](#2-프로젝트-설정)
3. [버전 번호 정하기](#3-버전-번호-정하기)
4. [테스트 빌드: APK + IPA를 Release에](#4-테스트-빌드-apk--ipa를-release에)
5. [스토어 업로드: Play + App Store Connect](#5-스토어-업로드-play--app-store-connect)
6. [자주 막히는 곳](#6-자주-막히는-곳)

---

## 1. 러너 Mac 준비하기

Android와 iOS 준비물에 Flutter만 더하면 돼요.

| 도구 | 확인 방법 | 참고 |
|---|---|---|
| Flutter SDK | `flutter --version` | [설치 안내](https://docs.flutter.dev/get-started/install/macos) |
| JDK 17+, Android SDK | `flutter doctor` | [android.md 2번](android.md#2-러너-mac-준비하기) |
| Xcode | `xcodebuild -version` | [ios.md 2번](ios.md#2-준비물) |
| CocoaPods | `pod --version` | 플러그인을 쓰는 iOS 앱에 필요해요. `brew install cocoapods` |

`flutter doctor`를 실행해서 Android와 iOS 항목에 문제가 없는지 먼저 확인하세요.

### 러너가 flutter 명령어를 찾을 수 있게 하기

터미널에서는 `flutter`가 되는데 러너에서는 `flutter: command not found`가 나는 경우가 많아요. 러너가 쓰는 PATH는 러너 폴더의 `.path` 파일에 따로 저장되어 있기 때문이에요.

가장 쉬운 방법은 **flutter가 잘 되는 터미널에서** 지금 PATH를 그대로 저장하는 거예요.

```bash
cd ~/actions-runners/REPO
```

```bash
echo "$PATH" > .path
```

`JAVA_HOME`, `ANDROID_HOME`은 `.env`에 적어요. 방법은 [android.md 2번](android.md#2-러너-mac-준비하기)과 같아요. 적고 나면 러너를 다시 시작하세요.

```bash
./svc.sh stop && ./svc.sh start
```

> Flutter 버전을 워크플로마다 고정하고 싶다면 [`subosito/flutter-action`](https://github.com/subosito/flutter-action)을 써도 돼요. 러너 Mac에 설치된 Flutter를 쓰면 더 빠르고 간단해서, 이 문서에서는 설치된 Flutter를 그대로 써요.

---

## 2. 프로젝트 설정

Flutter 프로젝트 안의 `android/`, `ios/` 폴더에 네이티브 앱과 똑같이 설정하면 돼요.

### Android (`android/` 폴더)
- 서명 키 만들기: [android.md 3번](android.md#3-서명-키-만들기)
- `android/app/build.gradle.kts`에 서명 설정: [android.md 4번](android.md#4-gradle에-서명-설정하기)
  - 단, **`versionCode` 설정은 건너뛰세요.** Flutter는 버전을 `pubspec.yaml`과 빌드 옵션으로 정해요([3번](#3-버전-번호-정하기)).

### iOS (`ios/` 폴더)
- 인증서, 프로필, API 키 만들기: [ios.md 3번](ios.md#3-apple-developer에서-만들-것들)
- `ios/Runner.xcworkspace`를 Xcode로 열고 **Runner** 타깃의 Release 서명을 수동으로 설정: [ios.md 4번](ios.md#4-xcode-프로젝트-설정)
- `ios/ExportOptions-AdHoc.plist`, `ios/ExportOptions-AppStore.plist` 만들기: [ios.md 5번](ios.md#5-exportoptions-파일-만들기)

### GitHub Secrets
[android.md 5번](android.md#5-github-secrets-등록하기)과 [ios.md 6번](ios.md#6-github-secrets-등록하기)의 Secret을 모두 등록해 주세요. 이름을 똑같이 쓰면 아래 워크플로를 그대로 쓸 수 있어요.

---

## 3. 버전 번호 정하기

Flutter는 `pubspec.yaml`의 `version`으로 두 플랫폼의 버전을 한 번에 정해요.

```yaml
version: 1.0.0+1
#        ─┬───  ┬
#         │     └ 빌드 번호 → Android versionCode, iOS CFBundleVersion
#         └ 버전 이름 → Android versionName, iOS CFBundleShortVersionString
```

스토어에 올릴 때마다 **빌드 번호는 이전보다 커야** 해요. 매번 손으로 올리기 번거로우니, 워크플로에서 실행 번호로 덮어쓰면 편해요.

```bash
flutter build appbundle --release --build-number=${{ github.run_number }}
```

버전 이름(`1.0.0`)은 `pubspec.yaml`에서 직접 관리하고, 태그(`v1.0.0`)와 맞춰 두면 헷갈리지 않아요.

---

## 4. 테스트 빌드: APK + IPA를 Release에

Actions 탭에서 버튼을 누르면 **APK와 Ad Hoc IPA를 만들어서 Release 하나에** 올려요.

`.github/workflows/flutter-test.yml`:

```yaml
name: Flutter 테스트 빌드

on:
  workflow_dispatch:

permissions:
  contents: write

jobs:
  build:
    runs-on: [self-hosted, macOS]
    env:
      KEYCHAIN: ${{ runner.temp }}/build.keychain-db
    steps:
      - uses: actions/checkout@v7

      - run: flutter pub get

      # ---------- Android ----------
      - name: keystore 꺼내기
        run: echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 --decode > "$RUNNER_TEMP/upload.jks"

      - name: APK 빌드
        env:
          ANDROID_KEYSTORE_PATH: ${{ runner.temp }}/upload.jks
          ANDROID_KEYSTORE_PASSWORD: ${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
          ANDROID_KEY_ALIAS: ${{ secrets.ANDROID_KEY_ALIAS }}
          ANDROID_KEY_PASSWORD: ${{ secrets.ANDROID_KEY_PASSWORD }}
        run: flutter build apk --release --build-number=${{ github.run_number }}

      # ---------- iOS ----------
      - name: iOS 서명 준비 (임시 키체인 + Ad Hoc 프로필)
        env:
          P12_BASE64: ${{ secrets.IOS_DIST_CERT_P12_BASE64 }}
          P12_PASSWORD: ${{ secrets.IOS_DIST_CERT_PASSWORD }}
          PROFILE_BASE64: ${{ secrets.IOS_ADHOC_PROFILE_BASE64 }}
        run: |
          KEYCHAIN_PASSWORD=$(openssl rand -base64 24)
          security create-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN"
          security set-keychain-settings -lut 21600 "$KEYCHAIN"
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN"

          echo "$P12_BASE64" | base64 --decode > "$RUNNER_TEMP/dist.p12"
          security import "$RUNNER_TEMP/dist.p12" -P "$P12_PASSWORD" -A -t cert -f pkcs12 -k "$KEYCHAIN"
          security set-key-partition-list -S apple-tool:,apple:,codesign: -s -k "$KEYCHAIN_PASSWORD" "$KEYCHAIN"
          security list-keychains -d user -s "$KEYCHAIN" $(security list-keychains -d user | tr -d '"')
          rm -f "$RUNNER_TEMP/dist.p12"

          PROFILE_DIR="$HOME/Library/Developer/Xcode/UserData/Provisioning Profiles"
          mkdir -p "$PROFILE_DIR"
          echo "$PROFILE_BASE64" | base64 --decode > "$RUNNER_TEMP/profile.mobileprovision"
          UUID=$(security cms -D -i "$RUNNER_TEMP/profile.mobileprovision" | plutil -extract UUID raw -o - -)
          cp "$RUNNER_TEMP/profile.mobileprovision" "$PROFILE_DIR/$UUID.mobileprovision"

      - name: Ad Hoc IPA 빌드
        run: |
          flutter build ipa --release \
            --build-number=${{ github.run_number }} \
            --export-options-plist=ios/ExportOptions-AdHoc.plist

      # ---------- Release ----------
      - name: Release에 올리기
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh release create "test-${{ github.run_number }}" \
            build/app/outputs/flutter-apk/app-release.apk \
            build/ios/ipa/*.ipa \
            --title "테스트 빌드 #${{ github.run_number }}" \
            --prerelease \
            --notes "커밋: ${{ github.sha }}"

      - name: 정리
        if: always()
        run: |
          rm -f "$RUNNER_TEMP/upload.jks"
          security delete-keychain "$KEYCHAIN" || true
```

Release 페이지에서 받는 방법은 여기를 보세요.
- APK: [android.md "휴대폰에 설치하기"](android.md#휴대폰에-설치하기)
- IPA: [ios.md "아이폰에 설치하기"](ios.md#아이폰에-설치하기)

> iOS 테스터에게는 Ad Hoc IPA보다 **TestFlight**가 편할 때가 많아요. 5번의 스토어 업로드를 하면 TestFlight에도 자동으로 올라가요.

---

## 5. 스토어 업로드: Play + App Store Connect

태그를 push하면 **AAB는 Play Console 내부 테스트로, IPA는 App Store Connect로** 올려요. Android와 iOS를 job 두 개로 나눠서, 한쪽이 실패해도 다른 쪽은 그대로 올라가게 했어요. 러너가 하나라면 두 job은 차례로 실행돼요.

처음 한 번 해 둘 일이 있어요.
- Play: 첫 AAB는 Play Console에서 직접 올리고, 서비스 계정을 연결해 두세요 ([android.md 7-1](android.md#7-1-처음-한-번만-해-둘-일)).
- App Store: App Store Connect에 앱을 만들어 두세요 ([ios.md 2번](ios.md#2-준비물)).

`.github/workflows/flutter-release.yml`:

```yaml
name: Flutter 스토어 업로드

on:
  push:
    tags: ['v*']

jobs:
  android:
    runs-on: [self-hosted, macOS]
    steps:
      - uses: actions/checkout@v7

      - run: flutter pub get

      - name: keystore 꺼내기
        run: echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 --decode > "$RUNNER_TEMP/upload.jks"

      - name: AAB 빌드
        env:
          ANDROID_KEYSTORE_PATH: ${{ runner.temp }}/upload.jks
          ANDROID_KEYSTORE_PASSWORD: ${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
          ANDROID_KEY_ALIAS: ${{ secrets.ANDROID_KEY_ALIAS }}
          ANDROID_KEY_PASSWORD: ${{ secrets.ANDROID_KEY_PASSWORD }}
        run: flutter build appbundle --release --build-number=${{ github.run_number }}

      - name: Play Console에 올리기
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}
          packageName: com.example.app      # 내 앱의 패키지 이름
          releaseFiles: build/app/outputs/bundle/release/app-release.aab
          track: internal
          status: completed

      - name: 정리
        if: always()
        run: rm -f "$RUNNER_TEMP/upload.jks"

  ios:
    runs-on: [self-hosted, macOS]
    env:
      KEYCHAIN: ${{ runner.temp }}/build.keychain-db
    steps:
      - uses: actions/checkout@v7

      - run: flutter pub get

      - name: iOS 서명 준비 (임시 키체인 + App Store 프로필)
        env:
          P12_BASE64: ${{ secrets.IOS_DIST_CERT_P12_BASE64 }}
          P12_PASSWORD: ${{ secrets.IOS_DIST_CERT_PASSWORD }}
          PROFILE_BASE64: ${{ secrets.IOS_APPSTORE_PROFILE_BASE64 }}
        run: |
          KEYCHAIN_PASSWORD=$(openssl rand -base64 24)
          security create-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN"
          security set-keychain-settings -lut 21600 "$KEYCHAIN"
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN"

          echo "$P12_BASE64" | base64 --decode > "$RUNNER_TEMP/dist.p12"
          security import "$RUNNER_TEMP/dist.p12" -P "$P12_PASSWORD" -A -t cert -f pkcs12 -k "$KEYCHAIN"
          security set-key-partition-list -S apple-tool:,apple:,codesign: -s -k "$KEYCHAIN_PASSWORD" "$KEYCHAIN"
          security list-keychains -d user -s "$KEYCHAIN" $(security list-keychains -d user | tr -d '"')
          rm -f "$RUNNER_TEMP/dist.p12"

          PROFILE_DIR="$HOME/Library/Developer/Xcode/UserData/Provisioning Profiles"
          mkdir -p "$PROFILE_DIR"
          echo "$PROFILE_BASE64" | base64 --decode > "$RUNNER_TEMP/profile.mobileprovision"
          UUID=$(security cms -D -i "$RUNNER_TEMP/profile.mobileprovision" | plutil -extract UUID raw -o - -)
          cp "$RUNNER_TEMP/profile.mobileprovision" "$PROFILE_DIR/$UUID.mobileprovision"

      - name: Archive
        run: |
          # Flutter 설정(버전, 플러그인)만 준비하고, archive는 xcodebuild로 직접 해요
          flutter build ios --release --config-only --build-number=${{ github.run_number }}
          xcodebuild archive \
            -workspace ios/Runner.xcworkspace \
            -scheme Runner \
            -configuration Release \
            -destination 'generic/platform=iOS' \
            -archivePath "$RUNNER_TEMP/Runner.xcarchive"

      - name: App Store Connect에 업로드
        env:
          ASC_KEY_P8_BASE64: ${{ secrets.ASC_KEY_P8_BASE64 }}
        run: |
          echo "$ASC_KEY_P8_BASE64" | base64 --decode > "$RUNNER_TEMP/AuthKey.p8"
          xcodebuild -exportArchive \
            -archivePath "$RUNNER_TEMP/Runner.xcarchive" \
            -exportOptionsPlist ios/ExportOptions-AppStore.plist \
            -exportPath "$RUNNER_TEMP/export" \
            -allowProvisioningUpdates \
            -authenticationKeyPath "$RUNNER_TEMP/AuthKey.p8" \
            -authenticationKeyID "${{ secrets.ASC_KEY_ID }}" \
            -authenticationKeyIssuerID "${{ secrets.ASC_ISSUER_ID }}"

      - name: 정리
        if: always()
        run: |
          rm -f "$RUNNER_TEMP/AuthKey.p8"
          security delete-keychain "$KEYCHAIN" || true
```

> **iOS는 왜 `flutter build ipa`를 안 쓰나요?**
> `flutter build ipa`는 export까지 한 번에 하는데, App Store용 ExportOptions에는 `destination: upload`가 들어 있어서 API 키 없이 업로드를 시도하다 실패해요. 그래서 `--config-only`로 Flutter 설정만 준비하고, archive와 업로드는 `xcodebuild`로 직접 해요.

```bash
git tag v1.0.0 && git push origin v1.0.0
```

업로드 뒤에 할 일은 네이티브 앱과 같아요.
- Play: [android.md 7-3](android.md#7-3-업로드-후에는)
- App Store: [ios.md "업로드 후에는"](ios.md#업로드-후에는)

---

## 6. 자주 막히는 곳

| 증상 | 이렇게 해 보세요 |
|---|---|
| `flutter: command not found` | 러너 폴더의 `.path`에 flutter 경로가 없어요. [1번](#러너가-flutter-명령어를-찾을-수-있게-하기)처럼 PATH를 저장하고 러너를 다시 시작하세요 |
| `pod: command not found` 또는 CocoaPods 오류 | `brew install cocoapods`로 설치하고, `.path`를 다시 저장하세요 |
| `Error: No valid code signing certificates were found` | iOS 서명 준비 단계가 실패했을 수 있어요. [ios.md 9번](ios.md#9-자주-막히는-곳)을 참고하세요 |
| Android는 되는데 iOS만 실패 | `ios/Runner.xcworkspace`에서 Release 서명을 수동으로 바꿨는지 확인하세요 |
| 로컬 Flutter와 러너 결과가 다름 | 러너 Mac의 `flutter --version`이 개발할 때 쓰는 버전과 같은지 확인하세요 |
| 두 번째 빌드부터 이상한 오류 | 러너는 작업 폴더를 지우지 않아요. 워크플로에 `flutter clean` 단계를 잠깐 넣어서 확인해 보세요 |

그 밖의 문제는 플랫폼별 문서를 참고하세요.
- [android.md 8번](android.md#8-자주-막히는-곳)
- [ios.md 9번](ios.md#9-자주-막히는-곳)

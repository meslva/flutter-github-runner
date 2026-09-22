# iOS: 테스트용 IPA부터 App Store 업로드까지

self-hosted runner로 iOS 앱을 빌드하는 방법이에요. iOS 빌드는 **macOS와 Xcode가 있어야만** 할 수 있어서, Mac을 러너로 쓰는 게 가장 큰 힘을 발휘하는 부분이에요.

이 문서를 따라 하면 두 가지를 할 수 있어요.

- **테스트용 IPA (Ad Hoc)**: 빌드해서 GitHub Release에 올리고, 등록된 아이폰에 설치
- **스토어 배포용 업로드**: App Store Connect에 자동 업로드 → TestFlight → App Store 심사

> 러너 설치가 아직이라면 [README](../README.md)부터 보고 오세요.
> Flutter 프로젝트라면 이 문서와 함께 [flutter.md](flutter.md)도 참고하세요. 서명과 업로드 방법은 같고, 빌드 명령어만 달라요.

## 목차
1. [전체 흐름 먼저 보기](#1-전체-흐름-먼저-보기)
2. [준비물](#2-준비물)
3. [Apple Developer에서 만들 것들](#3-apple-developer에서-만들-것들)
4. [Xcode 프로젝트 설정](#4-xcode-프로젝트-설정)
5. [ExportOptions 파일 만들기](#5-exportoptions-파일-만들기)
6. [GitHub Secrets 등록하기](#6-github-secrets-등록하기)
7. [테스트용 IPA 만들기 (Ad Hoc)](#7-테스트용-ipa-만들기-ad-hoc)
8. [App Store Connect에 올리기](#8-app-store-connect에-올리기)
9. [자주 막히는 곳](#9-자주-막히는-곳)

---

## 1. 전체 흐름 먼저 보기

iOS는 Android보다 준비할 게 많아요. 대신 한 번 설정해 두면 그다음부터는 버튼 하나로 끝나요.

```
archive (xcodebuild archive)
   │   소스를 빌드해서 .xcarchive를 만들어요
   ▼
export (xcodebuild -exportArchive)
   ├─ Ad Hoc으로 export        → 테스트용 .ipa → GitHub Release
   └─ App Store용으로 export    → App Store Connect에 바로 업로드
                                     ↓
                                 TestFlight → App Store 심사 → 출시
```

**archive는 한 번, export는 용도에 따라** 한다고 기억하면 쉬워요.

### 테스트 방법 두 가지

| | Ad Hoc IPA | TestFlight |
|---|---|---|
| 설치할 수 있는 기기 | 미리 등록한 기기만 (UDID, 연 100대) | 초대한 테스터 누구나 |
| 설치 방법 | Finder나 Apple Configurator로 설치 | TestFlight 앱에서 바로 설치 |
| Apple 심사 | 없음 | 내부 테스터는 없음, 외부 테스터는 간단한 심사 |
| 이 문서에서 | [7번](#7-테스트용-ipa-만들기-ad-hoc) | [8번](#8-app-store-connect에-올리기) 업로드 후 바로 사용 |

팀원 몇 명이 빠르게 확인하는 용도라면 **TestFlight가 훨씬 편해요.** Ad Hoc은 TestFlight를 쓸 수 없거나 파일로 보관해야 할 때 쓰면 좋아요.

---

## 2. 준비물

| 항목 | 설명 |
|---|---|
| Xcode | App Store 앱으로 설치해요. 설치 후 한 번 실행해서 추가 구성요소를 설치하세요 |
| Command Line Tools | `xcode-select -p`로 경로가 나오면 준비된 거예요 |
| Apple Developer Program | 유료 멤버십(연 $99)이 필요해요. Ad Hoc과 App Store 배포 모두 해당돼요 |
| App Store Connect 앱 | 스토어 업로드를 하려면 App Store Connect에 앱을 미리 만들어 둬야 해요 |

> App Store에 올리려면 Apple이 요구하는 최소 Xcode 버전이 있어요. 매년 기준이 올라가니 러너 Mac의 Xcode를 제때 업데이트해 주세요.

---

## 3. Apple Developer에서 만들 것들

[Apple Developer](https://developer.apple.com/account)의 **Certificates, Identifiers & Profiles**에서 만들어요.

### 3-1. App ID
**Identifiers**에서 앱의 번들 ID(예: `com.example.app`)를 등록해요. 이미 있다면 넘어가세요.

### 3-2. 배포용 인증서 (Apple Distribution)

Ad Hoc과 App Store 모두 **같은 Apple Distribution 인증서**를 써요.

1. Mac에서 **키체인 접근** 앱 → 메뉴 **인증서 지원** → **인증 기관에서 인증서 요청**으로 CSR 파일을 만들어요.
2. Apple Developer의 **Certificates**에서 **+** → **Apple Distribution**을 고르고 CSR을 올려요.
3. 받은 `.cer` 파일을 더블클릭해서 키체인에 설치해요.
4. 키체인 접근에서 인증서를 오른쪽 클릭 → **내보내기**로 `.p12` 파일을 만들어요. 이때 정하는 비밀번호를 기록해 두세요.

> `.p12`에는 개인 키가 들어 있어요. 저장소에 커밋하지 말고 안전한 곳에 보관하세요.

### 3-3. 프로비저닝 프로필 두 개

| 프로필 | 만드는 곳 | 필요한 것 |
|---|---|---|
| Ad Hoc | **Profiles** → **+** → **Ad Hoc** | App ID, 배포 인증서, **테스트할 기기** |
| App Store | **Profiles** → **+** → **App Store Connect** | App ID, 배포 인증서 |

Ad Hoc 프로필에는 설치할 아이폰이 미리 등록되어 있어야 해요. 기기는 **Devices**에서 UDID로 등록해요. 아이폰을 Mac에 연결하고 Finder에서 기기 이름 아래 정보를 클릭하면 UDID가 보여요.

> 기기를 새로 추가했다면 **Ad Hoc 프로필을 다시 받아서** Secret도 바꿔 줘야 해요. 프로필 안에 기기 목록이 들어 있기 때문이에요.

프로필 이름(예: `MyApp AdHoc`, `MyApp AppStore`)은 [5번](#5-exportoptions-파일-만들기)에서 쓰니 기억해 두세요.

### 3-4. App Store Connect API 키

업로드할 때 Apple ID로 로그인하는 대신 이 키를 써요.

1. [App Store Connect](https://appstoreconnect.apple.com/) → **사용자 및 액세스** → **통합** → **App Store Connect API**로 가요.
2. **팀 키**에서 새 키를 만들어요. 역할은 **App Manager**면 충분해요.
3. `.p8` 파일을 받고, 화면에 보이는 **Key ID**와 **Issuer ID**를 기록해요.

> `.p8` 파일은 **한 번만** 받을 수 있어요. 잃어버리면 새로 만들어야 해요.

---

## 4. Xcode 프로젝트 설정

CI에서는 서명을 **수동(manual)**으로 지정해 두는 게 가장 안정적이에요.

1. Xcode에서 프로젝트를 열고 앱 타깃 → **Signing & Capabilities**로 가요.
2. **Release** 설정에서 **Automatically manage signing**을 꺼요.
3. **Provisioning Profile**에서 App Store 프로필을 골라요.
4. Debug 설정은 그대로 자동 서명을 써도 괜찮아요. 내 컴퓨터에서 개발할 때 편해요.

### 수출 규정 질문 건너뛰기 (추천)

TestFlight에 올릴 때마다 "암호화를 사용하나요?"라는 질문에 답해야 하는데, `Info.plist`에 아래 값을 넣어 두면 건너뛸 수 있어요. 표준 HTTPS만 쓰는 대부분의 앱은 `false`예요.

```xml
<key>ITSAppUsesNonExemptEncryption</key>
<false/>
```

---

## 5. ExportOptions 파일 만들기

export할 때 "어떤 방식으로, 어떤 프로필로" 내보낼지 적는 파일이에요. 저장소에 두 개를 만들어요. `teamID`, 번들 ID, 프로필 이름은 내 값으로 바꿔 주세요. Team ID는 Apple Developer의 **Membership** 페이지에서 확인할 수 있어요.

`ios/ExportOptions-AdHoc.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>release-testing</string>
    <key>teamID</key>
    <string>ABCDE12345</string>
    <key>signingStyle</key>
    <string>manual</string>
    <key>signingCertificate</key>
    <string>Apple Distribution</string>
    <key>provisioningProfiles</key>
    <dict>
        <key>com.example.app</key>
        <string>MyApp AdHoc</string>
    </dict>
</dict>
</plist>
```

`ios/ExportOptions-AppStore.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store-connect</string>
    <key>destination</key>
    <string>upload</string>
    <key>manageAppVersionAndBuildNumber</key>
    <false/>
    <key>teamID</key>
    <string>ABCDE12345</string>
    <key>signingStyle</key>
    <string>manual</string>
    <key>signingCertificate</key>
    <string>Apple Distribution</string>
    <key>provisioningProfiles</key>
    <dict>
        <key>com.example.app</key>
        <string>MyApp AppStore</string>
    </dict>
</dict>
</plist>
```

- `destination: upload`: export와 동시에 App Store Connect로 올려요. 따로 업로드 도구가 필요 없어요.
- `manageAppVersionAndBuildNumber: false`: Xcode가 빌드 번호를 마음대로 바꾸지 않게 해요. 번호는 워크플로에서 직접 정해요.
- 앱 익스텐션(위젯 등)이 있다면 `provisioningProfiles`에 익스텐션의 번들 ID와 프로필도 추가해야 해요.

> `release-testing`과 `app-store-connect`는 Xcode 15.3부터 쓰는 이름이에요. 그보다 오래된 Xcode라면 각각 `ad-hoc`, `app-store`로 쓰세요.

---

## 6. GitHub Secrets 등록하기

저장소의 **Settings → Secrets and variables → Actions**에 등록해요.

| 이름 | 값 |
|---|---|
| `IOS_DIST_CERT_P12_BASE64` | 배포 인증서 `.p12`를 base64로 바꾼 값 |
| `IOS_DIST_CERT_PASSWORD` | `.p12` 비밀번호 |
| `IOS_ADHOC_PROFILE_BASE64` | Ad Hoc 프로필(`.mobileprovision`)을 base64로 바꾼 값 |
| `IOS_APPSTORE_PROFILE_BASE64` | App Store 프로필을 base64로 바꾼 값 |
| `ASC_KEY_ID` | App Store Connect API Key ID |
| `ASC_ISSUER_ID` | App Store Connect Issuer ID |
| `ASC_KEY_P8_BASE64` | API 키 `.p8`를 base64로 바꾼 값 |

`gh` CLI로 등록하면 편해요.

```bash
base64 -i dist.p12 | gh secret set IOS_DIST_CERT_P12_BASE64 --repo OWNER/REPO
```

```bash
base64 -i MyApp_AdHoc.mobileprovision | gh secret set IOS_ADHOC_PROFILE_BASE64 --repo OWNER/REPO
```

```bash
base64 -i MyApp_AppStore.mobileprovision | gh secret set IOS_APPSTORE_PROFILE_BASE64 --repo OWNER/REPO
```

```bash
base64 -i AuthKey_XXXXXXXXXX.p8 | gh secret set ASC_KEY_P8_BASE64 --repo OWNER/REPO
```

비밀번호와 ID는 값을 주지 않고 실행하면 입력창이 떠요.

```bash
gh secret set IOS_DIST_CERT_PASSWORD --repo OWNER/REPO
```

---

## 7. 테스트용 IPA 만들기 (Ad Hoc)

### 왜 임시 키체인을 쓰나요?

러너가 서명할 때 "키체인 접근을 허용하시겠습니까?" 팝업이 뜨면, 아무도 누르지 않으니 빌드가 그대로 멈춰 버려요. 그래서 워크플로에서 **빌드 전용 임시 키체인**을 만들고, 인증서를 넣고, 잠금을 풀어서 써요. 빌드가 끝나면 지워요.

> 러너 Mac의 로그인 키체인에 인증서를 직접 설치해 두고 쓰는 방법도 있어요. 이 경우 처음 한 번 팝업에서 **항상 허용**을 눌러 두면 되지만, 인증서를 바꿀 때마다 Mac에서 직접 손봐야 해서 이 문서에서는 임시 키체인 방식을 써요.

### 워크플로

`.github/workflows/ios-test.yml` (`MyApp`은 내 프로젝트 이름과 scheme으로 바꿔 주세요):

```yaml
name: iOS 테스트 빌드

on:
  workflow_dispatch:

permissions:
  contents: write

jobs:
  ipa:
    runs-on: [self-hosted, macOS]
    env:
      KEYCHAIN: ${{ runner.temp }}/build.keychain-db
    steps:
      - uses: actions/checkout@v7

      - name: 서명 준비 (임시 키체인 + 프로필)
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

      - name: Archive
        run: |
          xcodebuild archive \
            -project MyApp.xcodeproj \
            -scheme MyApp \
            -configuration Release \
            -destination 'generic/platform=iOS' \
            -archivePath "$RUNNER_TEMP/MyApp.xcarchive" \
            CURRENT_PROJECT_VERSION=${{ github.run_number }}

      - name: Ad Hoc IPA로 export
        run: |
          xcodebuild -exportArchive \
            -archivePath "$RUNNER_TEMP/MyApp.xcarchive" \
            -exportOptionsPlist ios/ExportOptions-AdHoc.plist \
            -exportPath "$RUNNER_TEMP/export"

      - name: Release에 올리기
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh release create "ios-test-${{ github.run_number }}" \
            "$RUNNER_TEMP"/export/*.ipa \
            --title "iOS 테스트 빌드 #${{ github.run_number }}" \
            --prerelease \
            --notes "커밋: ${{ github.sha }}"

      - name: 임시 키체인 지우기
        if: always()
        run: security delete-keychain "$KEYCHAIN" || true
```

- CocoaPods를 쓰는 프로젝트라면 Archive 전에 `pod install`을 실행하고, `-project` 대신 `-workspace MyApp.xcworkspace`를 쓰세요.
- `CURRENT_PROJECT_VERSION`으로 빌드 번호를 넣어요. `Info.plist`의 빌드 번호가 `$(CURRENT_PROJECT_VERSION)`으로 되어 있어야 반영돼요. 최근 Xcode 프로젝트는 기본으로 이렇게 되어 있어요.

### 아이폰에 설치하기

받은 `.ipa`는 아이폰에서 탭해서 바로 설치할 수는 없어요. Mac을 거쳐서 설치해요.

1. Release 페이지에서 `.ipa`를 Mac으로 받아요.
2. 아이폰을 Mac에 USB로 연결하고 Finder에서 아이폰을 선택해요.
3. `.ipa` 파일을 Finder 창의 아이폰 화면으로 끌어다 놓으면 설치돼요.

Apple Configurator 앱을 써도 돼요. Ad Hoc 프로필에 등록된 기기에만 설치되니, 설치가 안 되면 먼저 기기 등록부터 확인하세요.

---

## 8. App Store Connect에 올리기

태그를 push하면 빌드해서 App Store Connect에 올리는 워크플로예요. 올라간 빌드는 **TestFlight에서 바로 테스트**할 수 있고, 그대로 **App Store 심사에 제출**할 수도 있어요.

`.github/workflows/ios-release.yml`:

```yaml
name: iOS 스토어 업로드

on:
  push:
    tags: ['v*']

jobs:
  upload:
    runs-on: [self-hosted, macOS]
    env:
      KEYCHAIN: ${{ runner.temp }}/build.keychain-db
    steps:
      - uses: actions/checkout@v7

      - name: 서명 준비 (임시 키체인 + 프로필)
        env:
          P12_BASE64: ${{ secrets.IOS_DIST_CERT_P12_BASE64 }}
          P12_PASSWORD: ${{ secrets.IOS_DIST_CERT_PASSWORD }}
          PROFILE_BASE64: ${{ secrets.IOS_APPSTORE_PROFILE_BASE64 }}
        run: |
          # 7번의 "서명 준비" 단계와 똑같아요 (프로필 Secret만 App Store용)
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
          xcodebuild archive \
            -project MyApp.xcodeproj \
            -scheme MyApp \
            -configuration Release \
            -destination 'generic/platform=iOS' \
            -archivePath "$RUNNER_TEMP/MyApp.xcarchive" \
            CURRENT_PROJECT_VERSION=${{ github.run_number }}

      - name: App Store Connect에 업로드
        env:
          ASC_KEY_P8_BASE64: ${{ secrets.ASC_KEY_P8_BASE64 }}
        run: |
          echo "$ASC_KEY_P8_BASE64" | base64 --decode > "$RUNNER_TEMP/AuthKey.p8"
          xcodebuild -exportArchive \
            -archivePath "$RUNNER_TEMP/MyApp.xcarchive" \
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

```bash
git tag v1.0.0 && git push origin v1.0.0
```

### 업로드 후에는

1. App Store Connect에서 Apple이 빌드를 처리해요. 보통 수 분에서 수십 분 걸리고, 끝나면 메일이 와요.
2. **TestFlight** 탭에 빌드가 나타나요.
   - 내부 테스터(팀 멤버)는 심사 없이 바로 설치할 수 있어요.
   - 외부 테스터는 첫 빌드에 간단한 심사가 있어요.
3. 출시할 준비가 되면 **App Store** 탭에서 버전을 만들고, 빌드를 고른 뒤 **심사에 제출**해요.

> 심사 제출까지 자동화하고 싶다면 [fastlane](https://docs.fastlane.tools/)의 `deliver`를 쓰는 방법이 있어요. 스크린샷, 설명 문구 관리까지 한 번에 할 수 있지만, 처음에는 제출은 웹에서 직접 하는 걸 추천해요.

---

## 9. 자주 막히는 곳

| 증상 | 이렇게 해 보세요 |
|---|---|
| 빌드가 서명 단계에서 멈춤 | 키체인 팝업이 떴을 가능성이 커요. 임시 키체인 단계에서 `set-key-partition-list`가 실행됐는지 확인하세요 |
| `No signing certificate "Apple Distribution" found` | 인증서가 키체인에 안 들어갔어요. `.p12` 비밀번호와 base64 값이 맞는지 확인하세요 |
| `No profiles for 'com.example.app' were found` | 프로필 설치 위치나 번들 ID를 확인하세요. ExportOptions의 프로필 이름이 Apple Developer의 이름과 정확히 같아야 해요 |
| `requires a provisioning profile` (Archive 단계) | Xcode의 Release 서명 설정에서 프로필을 골랐는지 확인하세요 ([4번](#4-xcode-프로젝트-설정)) |
| 아이폰에 Ad Hoc IPA가 설치되지 않음 | 그 기기가 Ad Hoc 프로필에 등록되어 있는지 확인하세요. 기기를 추가했다면 프로필을 다시 받아야 해요 |
| 업로드에서 `The bundle version must be higher` | 빌드 번호가 이전 업로드보다 커야 해요. `CURRENT_PROJECT_VERSION`이 반영되는지 확인하세요 |
| 업로드에서 인증 오류 | `ASC_KEY_ID`, `ASC_ISSUER_ID`가 맞는지, API 키가 취소되지 않았는지 확인하세요 |
| 업로드에서 SDK 버전 오류 | Apple이 요구하는 최소 Xcode 버전보다 오래됐어요. 러너 Mac의 Xcode를 업데이트하세요 |
| 서명 오류가 나는데 원인을 모르겠음 | 러너 Mac에서 `security find-identity -v -p codesigning`을 실행해서 인증서가 보이는지 확인해 보세요 |

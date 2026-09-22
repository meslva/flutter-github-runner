# Android: 테스트용 APK부터 Play 스토어 업로드까지

self-hosted runner로 Android 앱을 빌드하는 방법이에요. 이 문서를 따라 하면 두 가지를 할 수 있어요.

- **테스트용 APK**: 빌드해서 GitHub Release에 올리고, 휴대폰에서 받아 바로 설치
- **스토어 배포용 AAB**: 서명해서 Google Play Console에 자동 업로드

> 러너 설치가 아직이라면 [README](../README.md)부터 보고 오세요.
> Flutter 프로젝트라면 이 문서와 함께 [flutter.md](flutter.md)도 참고하세요. 서명과 업로드 방법은 같고, 빌드 명령어만 달라요.

## 목차
1. [APK와 AAB, 뭐가 다른가요?](#1-apk와-aab-뭐가-다른가요)
2. [러너 Mac 준비하기](#2-러너-mac-준비하기)
3. [서명 키 만들기](#3-서명-키-만들기)
4. [Gradle에 서명 설정하기](#4-gradle에-서명-설정하기)
5. [GitHub Secrets 등록하기](#5-github-secrets-등록하기)
6. [테스트용 APK 만들기](#6-테스트용-apk-만들기)
7. [Play 스토어에 올리기](#7-play-스토어에-올리기)
8. [자주 막히는 곳](#8-자주-막히는-곳)

---

## 1. APK와 AAB, 뭐가 다른가요?

| | APK | AAB (App Bundle) |
|---|---|---|
| 한 줄 요약 | 바로 설치할 수 있는 앱 파일 | Play 스토어에 올리는 파일 |
| 휴대폰에 직접 설치 | 가능해요 | 안 돼요. Play가 기기에 맞는 APK로 바꿔서 배포해요 |
| 주로 쓰는 곳 | 테스트, 사내 배포 | Google Play 출시 (새 앱은 AAB만 받아요) |
| Gradle 명령어 | `./gradlew assembleRelease` | `./gradlew bundleRelease` |
| 결과 위치 | `app/build/outputs/apk/release/` | `app/build/outputs/bundle/release/` |

그래서 보통 **테스트는 APK, 출시는 AAB**로 나눠서 씁니다. 둘 다 같은 서명 키로 서명할 수 있어요.

---

## 2. 러너 Mac 준비하기

러너는 Mac에 설치된 도구로 빌드하기 때문에, 먼저 도구가 있어야 해요.

| 도구 | 설명 |
|---|---|
| JDK 17 이상 | Android Gradle Plugin 8.x부터 JDK 17이 필요해요. Android Studio에 들어 있는 JDK를 써도 돼요 |
| Android SDK | Android Studio를 설치하면 `~/Library/Android/sdk`에 생겨요 |

### 러너가 도구를 찾을 수 있게 하기

터미널에서는 빌드가 잘 되는데 러너에서는 `JAVA_HOME is not set` 같은 오류가 나는 경우가 많아요. 러너는 내 터미널 설정(`~/.zshrc`)을 읽지 않기 때문이에요.

러너 폴더의 `.env` 파일에 경로를 적어 두면 해결돼요.

```bash
cd ~/actions-runners/REPO
```

```bash
cat >> .env <<'EOF'
JAVA_HOME=/Applications/Android Studio.app/Contents/jbr/Contents/Home
ANDROID_HOME=/Users/사용자이름/Library/Android/sdk
EOF
```

- `JAVA_HOME`은 설치한 JDK 경로로 바꿔 주세요. Homebrew로 설치했다면 `/usr/libexec/java_home -v 17`로 경로를 확인할 수 있어요.
- `사용자이름`은 실제 macOS 사용자 이름으로 바꿔 주세요. `.env`에서는 `~`가 펼쳐지지 않아서 전체 경로를 써야 해요.

적은 뒤에는 러너를 다시 시작해야 반영돼요.

```bash
./svc.sh stop && ./svc.sh start
```

---

## 3. 서명 키 만들기

앱을 서명하려면 **keystore** 파일이 필요해요. 이미 쓰고 있는 keystore가 있다면 이 단계는 건너뛰세요.

```bash
keytool -genkeypair -v \
  -keystore upload-keystore.jks \
  -alias upload \
  -keyalg RSA -keysize 2048 -validity 10000
```

비밀번호와 이름 등을 물어보는데, 입력한 **keystore 비밀번호, alias(`upload`), key 비밀번호**를 꼭 기록해 두세요.

### alias를 빼먹고 만들었다면

`-alias` 없이 만들면 alias가 자동으로 `mykey`가 돼요. keystore에 들어 있는 alias는 이렇게 확인할 수 있어요.

```bash
keytool -list -keystore upload-keystore.jks
```

출력에서 `mykey, ...` 처럼 쉼표 앞에 있는 이름이 alias예요. 둘 중 하나를 고르면 돼요.

- **그대로 쓰기**: `ANDROID_KEY_ALIAS` Secret에 `mykey`를 넣으면 돼요.
- **이름 바꾸기**: 아래 명령으로 `upload`로 바꿀 수 있어요. 바꾸기 전에 keystore를 꼭 복사해 두세요.

```bash
keytool -changealias -keystore upload-keystore.jks -alias mykey -destalias upload
```

> 이미 이 키로 서명한 AAB를 Play에 올렸더라도 alias만 바꾸는 건 괜찮아요. 서명은 키 자체로 확인하고, alias는 keystore 안에서 키를 찾는 이름일 뿐이에요.

> **keystore는 꼭 안전한 곳에 백업해 두세요.**
> 잃어버리면 앱 업데이트가 어려워져요. 그리고 **절대 저장소에 커밋하지 마세요.** `.gitignore`에 `*.jks`, `*.keystore`를 넣어 두면 안심이에요.

### Play 스토어의 키 두 가지

Google Play를 쓰면 키가 두 종류로 나뉘어요.

- **업로드 키**: 방금 만든 keystore예요. AAB를 서명해서 Play에 올릴 때 써요.
- **앱 서명 키**: Google이 보관하는 키예요(Play App Signing). Play가 사용자에게 배포하는 앱은 이 키로 다시 서명돼요.

그래서 **Play에서 받은 앱과 Release에서 받은 테스트 APK는 서명이 달라요.** 한 휴대폰에서 둘을 덮어서 업데이트 설치할 수 없으니, 테스트할 때는 한쪽을 지우고 설치하세요.

---

## 4. Gradle에 서명 설정하기

서명 정보는 코드에 적지 않아요. 대신 **내 컴퓨터에서는 `key.properties` 파일**, **러너에서는 환경 변수(Secrets)**로 받도록 설정하면, 한 설정으로 두 곳에서 모두 빌드할 수 있어요.

```
환경 변수(ANDROID_KEYSTORE_PATH)가 있나?
  ├─ 있음 → 러너 방식: Secrets에서 꺼낸 keystore로 서명
  └─ 없음 → key.properties가 있나?
              ├─ 있음 → 로컬 방식: 파일에 적힌 keystore로 서명
              └─ 없음 → debug 키로 서명 (서명 정보가 없는 팀원도 빌드 가능)
```

### 4-1. 로컬용 파일 두기

이 문서에서는 `key.properties`와 keystore를 **모두 `app/` 폴더**(Flutter라면 `android/app/`)에 둬요.

```
android/                   (네이티브 프로젝트라면 프로젝트 루트)
├─ settings.gradle.kts
└─ app/
    ├─ build.gradle.kts
    ├─ key.properties      ← 로컬 서명 정보
    └─ upload-keystore.jks ← keystore
```

`app/key.properties`:

```properties
storePassword=keystore 비밀번호
keyPassword=key 비밀번호
keyAlias=upload
storeFile=upload-keystore.jks
```

- `storeFile`을 상대 경로로 쓰면 **`app/` 폴더 기준**으로 찾아요. 같은 폴더에 두었으니 파일 이름만 적으면 돼요.
- keystore를 프로젝트 밖(예: `~/keys/android/`)에 둔다면 `storeFile=/Users/사용자이름/keys/android/upload-keystore.jks`처럼 절대 경로로 적어요. `~`는 펼쳐지지 않아요.

> **두 파일은 절대 커밋하면 안 돼요.** `.gitignore`에 아래 줄이 있는지 확인하세요. Flutter가 만든 `android/.gitignore`에는 보통 들어 있어요.
>
> ```gitignore
> key.properties
> **/*.jks
> **/*.keystore
> ```
>
> 실제로 무시되는지는 `git check-ignore -v android/app/key.properties android/app/upload-keystore.jks`로 확인할 수 있어요. 두 경로가 출력되면 무시되고 있는 거예요.

### 4-2. build.gradle.kts 설정

`app/build.gradle.kts` (Flutter라면 `android/app/build.gradle.kts`):

```kotlin
import java.util.Properties

// 로컬: app/key.properties (없으면 빈 값)
val keystoreProperties = Properties().apply {
    val propertiesFile = file("key.properties")
    if (propertiesFile.exists()) propertiesFile.inputStream().use { load(it) }
}

// 러너의 환경 변수가 있으면 그 값을, 없으면 key.properties 값을 써요
fun signingValue(envName: String, propertyName: String): String? =
    System.getenv(envName) ?: keystoreProperties.getProperty(propertyName)

val keystorePath: String? = signingValue("ANDROID_KEYSTORE_PATH", "storeFile")

android {
    // ... 기존 설정 ...

    signingConfigs {
        if (keystorePath != null) {
            create("release") {
                storeFile = file(keystorePath)
                storePassword = signingValue("ANDROID_KEYSTORE_PASSWORD", "storePassword")
                keyAlias = signingValue("ANDROID_KEY_ALIAS", "keyAlias")
                keyPassword = signingValue("ANDROID_KEY_PASSWORD", "keyPassword")
            }
        }
    }

    buildTypes {
        release {
            signingConfig = if (keystorePath != null) {
                signingConfigs.getByName("release")
            } else {
                signingConfigs.getByName("debug")
            }
        }
    }
}
```

**코드를 넣는 위치**

- `import java.util.Properties`는 파일 **맨 위**에 둬요.
- `keystoreProperties`, `signingValue`, `keystorePath`는 **`plugins { }` 블록 다음, `android { }` 블록 앞**에 둬요. `.kts` 파일에서는 `plugins { }`가 import 다음 첫 블록이어야 해요.
- Flutter 프로젝트에는 `release { signingConfig = signingConfigs.getByName("debug") }`가 기본으로 들어 있어요. 이 부분을 위 `buildTypes` 내용으로 바꿔 주세요.

**파일 위치를 찾는 기준**

| 코드 | 기준 폴더 | 위 설정에서 가리키는 곳 |
|---|---|---|
| `file("key.properties")` | 이 `build.gradle.kts`가 있는 모듈 폴더 | `app/key.properties` |
| `rootProject.file("key.properties")` | 루트 프로젝트 폴더 (`settings.gradle.kts`가 있는 곳) | `key.properties` (Flutter라면 `android/key.properties`) |
| `file(keystorePath)` | 모듈 폴더. 절대 경로면 그대로 사용 | 로컬: `app/upload-keystore.jks` / 러너: `$RUNNER_TEMP/upload.jks` |

`key.properties`를 `android/` 폴더(루트)에 두고 싶다면 `file("key.properties")`를 `rootProject.file("key.properties")`로 바꾸면 돼요. Flutter 공식 문서가 이 위치를 써요.

### 4-3. 잘 되는지 확인하기

**로컬 (key.properties 사용)**: 환경 변수 없이 빌드하고 서명을 확인해요.

```bash
./gradlew assembleRelease
```

```bash
~/Library/Android/sdk/build-tools/<버전>/apksigner verify --print-certs app/build/outputs/apk/release/app-release.apk
```

Flutter라면 프로젝트 폴더에서 `flutter build apk --release`로 빌드하고, `build/app/outputs/flutter-apk/app-release.apk`를 확인하세요.

출력의 `certificate SHA-256 digest`가 내 keystore의 SHA256과 같으면 성공이에요. keystore 쪽 값은 `keytool -list -v -keystore upload-keystore.jks -alias upload`로 볼 수 있어요. `CN=Android Debug`가 보이면 `key.properties`를 못 읽고 debug 키로 서명된 거예요.

> `keytool -printcert -jarfile`은 예전 방식(v1) 서명만 읽어요. 요즘 APK는 v2 이상으로 서명되는 경우가 많아서 아무것도 안 나올 수 있으니 `apksigner`를 쓰세요.

**러너 (Secrets 사용)**: [6번](#6-테스트용-apk-만들기) 워크플로가 `ANDROID_KEYSTORE_PATH` 등을 환경 변수로 넘겨요. 러너가 checkout한 코드에는 `key.properties`가 없으니(커밋하지 않았으니까요) 자연스럽게 환경 변수 쪽이 쓰여요.

### 버전 코드 자동으로 올리기

Play에 올릴 때마다 `versionCode`가 이전보다 커야 해요. GitHub Actions의 실행 번호를 쓰면 매번 자동으로 올라가요.

```kotlin
android {
    defaultConfig {
        versionCode = System.getenv("BUILD_NUMBER")?.toInt() ?: 1
        versionName = "1.0.0"
    }
}
```

---

## 5. GitHub Secrets 등록하기

keystore와 비밀번호는 저장소의 **Settings → Secrets and variables → Actions**에 넣어 둬요. 워크플로에서만 꺼내 쓸 수 있고, 로그에도 가려져서 보여요.

| 이름 | 값 |
|---|---|
| `ANDROID_KEYSTORE_BASE64` | keystore 파일을 base64로 바꾼 문자열 |
| `ANDROID_KEYSTORE_PASSWORD` | keystore 비밀번호 |
| `ANDROID_KEY_ALIAS` | alias (예: `upload`) |
| `ANDROID_KEY_PASSWORD` | key 비밀번호 |
| `PLAY_SERVICE_ACCOUNT_JSON` | Play 업로드용 서비스 계정 키 ([7번](#7-play-스토어에-올리기)에서 만들어요) |

`gh` CLI를 쓰면 터미널에서 바로 등록할 수 있어요.

```bash
base64 -i upload-keystore.jks | gh secret set ANDROID_KEYSTORE_BASE64 --repo OWNER/REPO
```

```bash
gh secret set ANDROID_KEYSTORE_PASSWORD --repo OWNER/REPO
```

두 번째 명령처럼 값을 주지 않으면 입력창이 뜨고, 입력한 값이 화면에 보이지 않아요. 나머지도 같은 방식으로 등록하면 돼요.

---

## 6. 테스트용 APK 만들기

Actions 탭에서 버튼을 누르면 서명된 APK를 만들어서 **GitHub Release**에 올리는 워크플로예요.

`.github/workflows/android-test.yml`:

```yaml
name: Android 테스트 빌드

on:
  workflow_dispatch:   # Actions 탭에서 직접 실행

permissions:
  contents: write      # Release를 만들려면 필요해요

jobs:
  apk:
    runs-on: [self-hosted, macOS]
    steps:
      - uses: actions/checkout@v7

      - name: keystore 꺼내기
        run: echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 --decode > "$RUNNER_TEMP/upload.jks"

      - name: APK 빌드
        env:
          ANDROID_KEYSTORE_PATH: ${{ runner.temp }}/upload.jks
          ANDROID_KEYSTORE_PASSWORD: ${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
          ANDROID_KEY_ALIAS: ${{ secrets.ANDROID_KEY_ALIAS }}
          ANDROID_KEY_PASSWORD: ${{ secrets.ANDROID_KEY_PASSWORD }}
          BUILD_NUMBER: ${{ github.run_number }}
        run: ./gradlew assembleRelease

      - name: Release에 올리기
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh release create "android-test-${{ github.run_number }}" \
            app/build/outputs/apk/release/*.apk \
            --title "Android 테스트 빌드 #${{ github.run_number }}" \
            --prerelease \
            --notes "커밋: ${{ github.sha }}"

      - name: keystore 지우기
        if: always()
        run: rm -f "$RUNNER_TEMP/upload.jks"
```

### 휴대폰에 설치하기

1. 휴대폰 브라우저로 저장소의 **Releases** 페이지(`https://github.com/OWNER/REPO/releases`)에 들어가요.
2. 가장 위의 `Android 테스트 빌드`에서 `.apk` 파일을 눌러 받아요.
3. 처음이라면 "출처를 알 수 없는 앱 설치"를 허용해 달라는 안내가 나와요. 허용하고 설치하면 끝이에요.

> 비공개 저장소라면 휴대폰 브라우저에서도 GitHub에 로그인해야 파일을 받을 수 있어요.

**Release 말고 Artifact로 받고 싶다면**, 마지막 업로드 단계를 이렇게 바꾸면 돼요. 실행 기록 페이지에서 zip으로 받을 수 있고, 기본 90일 뒤에 자동으로 지워져요.

```yaml
      - uses: actions/upload-artifact@v7
        with:
          name: android-apk
          path: app/build/outputs/apk/release/*.apk
```

---

## 7. Play 스토어에 올리기

이번에는 태그를 push하면 AAB를 만들어서 **Play Console의 내부 테스트 트랙**에 올리는 워크플로예요.

### 7-1. 처음 한 번만 해 둘 일

**① 첫 번째 AAB는 직접 올려야 해요.**
Google Play API는 앱을 새로 만들 수 없어요. Play Console에서 앱을 만들고, 첫 AAB는 웹에서 한 번 직접 업로드해 주세요. 그다음부터는 자동으로 올릴 수 있어요.

**② 업로드용 서비스 계정 만들기**

1. [Google Cloud Console](https://console.cloud.google.com/)에서 프로젝트를 고르거나 새로 만들어요.
2. **Google Play Android Developer API**를 사용 설정해요.
3. **IAM 및 관리자 → 서비스 계정**에서 서비스 계정을 만들고, **키 → 키 추가 → JSON**으로 키 파일을 받아요.
4. [Play Console](https://play.google.com/console)의 **사용자 및 권한**에서 서비스 계정 이메일을 초대하고, 해당 앱의 **출시 관리** 권한을 줘요.
5. 받은 JSON 파일 내용을 `PLAY_SERVICE_ACCOUNT_JSON` Secret으로 등록해요.

```bash
gh secret set PLAY_SERVICE_ACCOUNT_JSON --repo OWNER/REPO < play-service-account.json
```

> 권한이 반영되기까지 시간이 조금 걸릴 수 있어요. 처음 업로드에서 권한 오류가 나면 잠시 뒤에 다시 실행해 보세요.

### 7-2. 워크플로

`.github/workflows/android-release.yml`:

```yaml
name: Android 스토어 업로드

on:
  push:
    tags: ['v*']       # 예: v1.0.0

jobs:
  aab:
    runs-on: [self-hosted, macOS]
    steps:
      - uses: actions/checkout@v7

      - name: keystore 꺼내기
        run: echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 --decode > "$RUNNER_TEMP/upload.jks"

      - name: AAB 빌드
        env:
          ANDROID_KEYSTORE_PATH: ${{ runner.temp }}/upload.jks
          ANDROID_KEYSTORE_PASSWORD: ${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
          ANDROID_KEY_ALIAS: ${{ secrets.ANDROID_KEY_ALIAS }}
          ANDROID_KEY_PASSWORD: ${{ secrets.ANDROID_KEY_PASSWORD }}
          BUILD_NUMBER: ${{ github.run_number }}
        run: ./gradlew bundleRelease

      - name: Play Console에 올리기
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}
          packageName: com.example.app      # 내 앱의 패키지 이름
          releaseFiles: app/build/outputs/bundle/release/*.aab
          track: internal                   # internal / alpha / beta / production
          status: completed

      - name: keystore 지우기
        if: always()
        run: rm -f "$RUNNER_TEMP/upload.jks"
```

태그를 붙여서 push하면 시작돼요.

```bash
git tag v1.0.0 && git push origin v1.0.0
```

### 7-3. 업로드 후에는

- **내부 테스트(`internal`)**: Play Console에 등록한 테스터가 몇 분 안에 Play 스토어에서 받을 수 있어요.
- **정식 출시**: 처음에는 `internal`로 올려서 확인한 뒤, Play Console에서 프로덕션으로 승격하는 방식을 추천해요. `track: production`으로 바로 올릴 수도 있지만, 실수하면 사용자에게 바로 나가니 조심하세요.

---

## 8. 자주 막히는 곳

| 증상 | 이렇게 해 보세요 |
|---|---|
| `JAVA_HOME is not set` / `SDK location not found` | 러너 폴더의 `.env`에 `JAVA_HOME`, `ANDROID_HOME`을 적고 러너를 다시 시작하세요 ([2번](#2-러너-mac-준비하기)) |
| `Unsupported class file major version` | JDK 버전이 맞지 않아요. `JAVA_HOME`이 JDK 17 이상을 가리키는지 확인하세요 |
| `Keystore was tampered with, or password was incorrect` | `ANDROID_KEYSTORE_PASSWORD`가 틀렸거나, base64 변환 중에 파일이 깨졌어요. Secret을 다시 등록해 보세요 |
| 휴대폰에서 "앱이 설치되지 않았습니다" | 서명이 다른 같은 앱이 이미 깔려 있을 수 있어요. 기존 앱을 지우고 다시 설치하세요 |
| Play 업로드에서 `Version code has already been used` | `versionCode`가 이전 업로드보다 커야 해요. `BUILD_NUMBER`가 제대로 들어가는지 확인하세요 |
| Play 업로드에서 `The caller does not have permission` | 서비스 계정을 Play Console에 초대했는지, 앱 권한을 줬는지 확인하세요. 반영에 시간이 걸릴 수도 있어요 |
| Play 업로드에서 `Package not found` | 첫 AAB를 Play Console에서 직접 올렸는지, `packageName`이 맞는지 확인하세요 |

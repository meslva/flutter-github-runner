# macOS에 GitHub Actions Self-hosted Runner 설치하기

내 Mac을 GitHub Actions의 **self-hosted runner**로 등록해서, 워크플로를 GitHub 서버 대신 내 컴퓨터에서 실행하는 방법을 정리한 문서입니다.

> 이 문서는 **개인 저장소 1개**에 러너를 등록하는 경우를 기준으로 합니다.
> 문서의 `OWNER/REPO`는 실제 저장소 이름(예: `octocat/hello-world`)으로 바꿔서 사용하세요.

## 목차
1. [개요](#1-개요)
2. [사전 준비](#2-사전-준비)
3. [설치](#3-설치)
4. [실행](#4-실행)
5. [워크플로에서 사용하기](#5-워크플로에서-사용하기)
6. [동작 확인](#6-동작-확인)
7. [보안 주의사항](#7-보안-주의사항)
8. [문제 해결](#8-문제-해결)
9. [제거 방법](#9-제거-방법)
10. [여러 저장소에서 쓰기](#10-여러-저장소에서-쓰기)

### 앱 빌드 가이드

러너를 설치했다면 앱을 빌드해 볼 차례예요. 테스트용 파일을 만들어 Release에 올리는 방법과 스토어에 업로드하는 방법을 다뤄요.

| 문서 | 내용 |
|---|---|
| [Android](docs/android.md) | 테스트용 APK → Release, AAB → Google Play |
| [iOS](docs/ios.md) | 테스트용 Ad Hoc IPA → Release, App Store Connect(TestFlight) 업로드 |
| [Flutter](docs/flutter.md) | 워크플로 하나로 APK, AAB, IPA를 한 번에 |

---

## 1. 개요

self-hosted runner는 내 컴퓨터에서 실행되는 작은 프로그램입니다. GitHub에 할 작업이 있는지 계속 물어보다가, 작업이 생기면 받아와서 내 컴퓨터에서 실행하고 결과를 GitHub에 올립니다.

```
[내 Mac의 러너]  ── HTTPS(443) 나가는 연결 ──▶  [GitHub]
      ▲                                           │
      └──── "할 작업 있어?" 물어보고 받아옴 ◀─────┘
```

- 연결은 항상 **내 컴퓨터에서 GitHub 쪽으로 나가는 방향**입니다. GitHub이 내 컴퓨터로 먼저 접속하지 않습니다.
- 그래서 공인 IP, 공유기 포트포워딩, 방화벽 포트 개방이 **필요 없습니다**.
- self-hosted runner에서 실행한 작업은 GitHub Actions 사용 시간(분)이 **차감되지 않습니다**.

---

## 2. 사전 준비

### 2-1. 하드웨어 / OS

| 항목 | 요구사항 |
|---|---|
| OS | macOS 11 (Big Sur) 이상 |
| CPU | Apple Silicon(ARM64) 또는 Intel(x64). CPU에 맞는 패키지를 받아야 합니다 |
| 디스크 | 러너 자체는 수백 MB입니다. 빌드 결과물과 캐시를 생각하면 여유 공간 10GB 이상을 권장합니다 |
| 메모리 | 최소 2GB. 실행할 빌드 작업에 따라 더 필요합니다 |

CPU 종류는 터미널에서 확인할 수 있습니다.

```bash
uname -m
```

- `arm64` → Apple Silicon → `osx-arm64` 패키지
- `x86_64` → Intel → `osx-x64` 패키지

### 2-2. 네트워크

- **들어오는 접속은 필요 없습니다.**
- **나가는 HTTPS(443) 연결**이 되어야 하고, 업로드·다운로드 속도는 최소 70kbps가 필요합니다.
- 최소한 아래 도메인에 접속할 수 있어야 합니다.

| 용도 | 도메인 |
|---|---|
| 기본 동작 | `github.com`, `api.github.com`, `*.actions.githubusercontent.com` |
| 액션 다운로드 | `codeload.github.com` |
| 아티팩트·로그·캐시 | `results-receiver.actions.githubusercontent.com`, `*.blob.core.windows.net` |
| 러너 자동 업데이트 | `objects.githubusercontent.com`, `objects-origin.githubusercontent.com`, `github-releases.githubusercontent.com`, `github-registry-files.githubusercontent.com` |

가정용 인터넷이라면 따로 설정할 것이 없습니다. 회사망처럼 외부 접속을 제한하거나 프록시를 쓰는 곳이라면 위 도메인을 허용하거나 `https_proxy` 환경 변수를 설정해야 합니다. 전체 목록은 [공식 문서](https://docs.github.com/en/actions/reference/runners/self-hosted-runners)에 있습니다.

### 2-3. GitHub 계정 / 권한

- 러너를 등록할 저장소의 **관리자(admin) 권한**이 필요합니다. 개인 저장소 소유자는 기본으로 가지고 있습니다.
- 등록할 때 쓰는 **등록 토큰은 1시간 뒤 만료**됩니다. 토큰을 받은 뒤 바로 설치를 진행하세요.
- 가능하면 **비공개(private) 저장소**에 연결하세요. 이유는 [7. 보안 주의사항](#7-보안-주의사항)을 참고하세요.

### 2-4. 로컬 소프트웨어

| 구분 | 도구 | 비고 |
|---|---|---|
| 필수 | `curl`, `tar` | macOS에 기본으로 들어 있습니다 |
| 권장 | `git` | 저장소 checkout에 사용합니다 |
| 권장 | [`gh` CLI](https://cli.github.com/) | 터미널에서 토큰을 발급할 때 사용합니다 (`brew install gh`) |
| 권장 | [Homebrew](https://brew.sh/) | 빌드 도구를 설치할 때 편합니다 |

> GitHub이 제공하는 러너와 달리 **self-hosted runner에는 도구가 미리 설치되어 있지 않습니다.**
> 워크플로에서 쓰는 Node.js, Python, Xcode 같은 도구는 직접 설치해야 합니다.
> Docker 컨테이너를 쓰는 작업(`container:`, `services:`, Docker 컨테이너 액션)은 Linux 러너에서만 동작하므로 macOS 러너에서는 쓸 수 없습니다.

### 2-5. 운영 조건

- 컴퓨터가 **켜져 있고 잠자기 상태가 아니어야** 작업을 받습니다. 잠자기 중에 들어온 작업은 대기열에서 기다립니다.
- 서비스로 등록하면 macOS의 **LaunchAgent**로 동작하므로 **사용자가 로그인해 있어야** 실행됩니다.
- 작업마다 깨끗한 환경이 새로 만들어지지 않습니다. 이전 작업이 남긴 파일과 설치한 도구가 그대로 남습니다.
- 러너는 **자동으로 업데이트**됩니다. 새 버전이 나온 뒤 30일 안에 업데이트하지 않으면 GitHub이 그 러너에 작업을 보내지 않습니다.
- 14일 넘게 GitHub에 연결되지 않은 러너는 자동으로 삭제됩니다. 이때는 다시 등록해야 합니다.

### 2-6. 체크리스트

- [ ] macOS 11 이상이고, CPU 종류(`arm64` / `x86_64`)를 확인했다
- [ ] 여유 디스크 공간이 충분하다
- [ ] 인터넷이 되고, GitHub 도메인으로 나가는 HTTPS 연결이 막혀 있지 않다
- [ ] 러너를 연결할 저장소의 관리자 권한이 있다 (가능하면 비공개 저장소)
- [ ] 워크플로에서 쓸 도구를 설치했거나 설치할 계획이 있다

---

## 3. 설치

### 한눈에 보는 설치 순서

위에서부터 차례로 따라 하면 됩니다. 명령어의 `OWNER/REPO`, `REPO`는 내 저장소 이름으로 바꾸세요.

| 단계 | 하는 일 | 어디서 |
|---|---|---|
| [3-1](#3-1-공개-저장소라면-보안-설정-먼저) | 공개 저장소라면 보안 설정 | 웹 |
| [3-2](#3-2-러너-폴더-만들기) | 러너 폴더 만들기 | 터미널 |
| [3-3](#3-3-러너-다운로드) | 러너 다운로드 | 터미널 |
| [3-4](#3-4-등록-토큰-발급) | 등록 토큰 발급 (1시간 유효) | 터미널 또는 웹 |
| [3-5](#3-5-러너-등록-configsh) | 러너 등록 | 터미널 |
| [4-1](#4-1-테스트-실행-runsh) | 테스트 실행, Idle 확인 | 터미널 + 웹 |
| [4-2](#4-2-상시-실행용-서비스로-등록) | 서비스로 등록 | 터미널 |
| [4-3](#4-3-빌드-도구-경로-설정) | 빌드 도구 경로 설정 | 터미널 |

### 3-1. 공개 저장소라면 보안 설정 먼저

비공개 저장소라면 건너뛰어도 됩니다. 공개 저장소라면 **러너를 붙이기 전에** 해 두세요.

1. 저장소 → **Settings** → **Actions** → **General**로 이동합니다.
2. **Approval for running fork pull request workflows from contributors**에서 **Require approval for all external contributors**를 고르고 **Save**를 누릅니다.

기본 설정은 "처음 기여하는 사람만 승인"이라서, 한 번이라도 기여한 사람의 PR은 승인 없이 내 Mac에서 실행됩니다. 자세한 이유는 [7. 보안 주의사항](#7-보안-주의사항)을 참고하세요.

### 3-2. 러너 폴더 만들기

러너 폴더는 **프로젝트(저장소) 폴더 밖에 따로** 만듭니다. 이 문서에서는 `~/actions-runners/` 아래에 **저장소 이름으로** 폴더를 만듭니다.

```bash
mkdir -p ~/actions-runners/REPO && cd ~/actions-runners/REPO
```

```
~/actions-runners/            ← 정리용 상위 폴더 (러너 아님)
├─ REPO/                      ← 러너 1: OWNER/REPO 저장소에 등록
└─ app2/                      ← 러너 2: 나중에 다른 저장소를 추가할 때
```

- **하위 폴더 하나가 러너 하나**이고, 러너 하나는 **저장소 하나**에 등록됩니다. 다른 저장소에서도 쓰려면 [10. 여러 저장소에서 쓰기](#10-여러-저장소에서-쓰기)를 참고하세요.
- 러너는 폴더 위치가 아니라 `config.sh`에 적은 **저장소 주소**로 연결됩니다. 빌드할 코드는 GitHub에서 새로 받아오므로, 내가 코드를 편집하는 프로젝트 폴더와는 관계가 없습니다.

**피해야 할 위치**

| 위치 | 이유 |
|---|---|
| 프로젝트(git 저장소) 폴더 안 | 러너 인증 정보(`.credentials` 등)가 커밋될 수 있고, `_work`에 저장소가 또 받아져서 꼬입니다 |
| Desktop, Documents, Downloads, iCloud Drive | macOS 개인정보 보호 대상이라 서비스로 실행될 때 권한 문제가 생기거나, 빌드 파일까지 동기화됩니다 |
| 경로에 공백이나 한글이 있는 곳 | 일부 빌드 스크립트가 이런 경로를 제대로 처리하지 못합니다 |

> **설치한 뒤에는 폴더를 옮기지 마세요.** 서비스 설정에 경로가 저장되어 있어서, 옮기려면 `./svc.sh uninstall` → 이동 → `./svc.sh install`을 다시 해야 합니다.

### 3-3. 러너 다운로드

3-2에서 만든 폴더 안에서 실행합니다. 최신 버전을 자동으로 찾아서 CPU에 맞는 패키지를 받습니다.

```bash
VERSION=$(curl -fsSL https://api.github.com/repos/actions/runner/releases/latest | grep '"tag_name"' | sed -E 's/.*"v([^"]+)".*/\1/')
ARCH=$([ "$(uname -m)" = "arm64" ] && echo arm64 || echo x64)
curl -fL -o actions-runner.tar.gz "https://github.com/actions/runner/releases/download/v${VERSION}/actions-runner-osx-${ARCH}-${VERSION}.tar.gz"
tar xzf actions-runner.tar.gz
```

`ls`를 실행했을 때 `config.sh`, `run.sh`, `svc.sh`가 보이면 된 것입니다.

> 저장소의 **Settings → Actions → Runners → New self-hosted runner** 화면에 나오는 `curl` / `tar` 명령어를 그대로 써도 결과는 같습니다. 그 화면에는 파일 해시(SHA-256) 확인 명령어도 있으니 함께 실행하면 더 안전합니다.

### 3-4. 등록 토큰 발급

등록 토큰은 **1시간 뒤 만료**됩니다. 그래서 다운로드를 마친 뒤, 등록하기 **바로 직전에** 발급받는 것이 좋습니다.

**방법 A. `gh` CLI로 발급 (추천)**

```bash
gh api -X POST repos/OWNER/REPO/actions/runners/registration-token --jq .token
```

출력된 문자열이 등록 토큰입니다. `gh`에 로그인되어 있지 않다면 먼저 `gh auth login`을 실행하세요.

**방법 B. 웹에서 발급**

1. 저장소 → **Settings** → **Actions** → **Runners** → **New self-hosted runner**를 누릅니다.
2. **macOS**와 CPU에 맞는 아키텍처(**ARM64** 또는 **x64**)를 선택합니다.
3. 화면의 `Configure` 부분에 있는 `--token` 값이 등록 토큰입니다.

### 3-5. 러너 등록 (config.sh)

`등록토큰`에 3-4에서 받은 값을 넣어 실행합니다.

```bash
./config.sh --url https://github.com/OWNER/REPO --token 등록토큰
```

실행하면 몇 가지를 물어봅니다. 모두 Enter를 누르면 기본값으로 진행됩니다.

| 질문 | 기본값 | 설명 |
|---|---|---|
| runner group | `Default` | 개인 저장소에서는 기본값을 쓰면 됩니다 |
| runner name | 컴퓨터 이름 | Runners 목록에 보이는 이름입니다 |
| additional labels | 없음 | 워크플로에서 이 러너를 고를 때 쓸 라벨입니다. 기본으로 `self-hosted`, `macOS`, `ARM64`(또는 `X64`)가 붙습니다 |
| work folder | `_work` | 작업이 실행되는 폴더입니다 |

질문 없이 한 번에 등록하려면 옵션을 함께 넘깁니다. 예를 들어 Flutter 빌드용 러너라면 `flutter` 라벨을 붙여 둘 수 있습니다.

```bash
./config.sh --url https://github.com/OWNER/REPO --token 등록토큰 --name my-mac --labels flutter --work _work --unattended
```

`√ Settings Saved.`가 나오면 등록이 끝난 것입니다.

---

## 4. 실행

### 4-1. 테스트 실행 (run.sh)

```bash
./run.sh
```

1. `Listening for Jobs`가 나오면 작업을 기다리는 상태입니다.
2. 저장소 → **Settings** → **Actions** → **Runners**에서 러너가 **Idle**(초록색)로 보이는지 확인합니다.
3. 확인했으면 `Ctrl+C`로 멈춥니다. `run.sh`는 터미널을 닫으면 같이 멈추므로 테스트 용도로만 씁니다.

### 4-2. 상시 실행용: 서비스로 등록

로그인할 때 자동으로 시작되고, 터미널을 닫아도 계속 실행되게 하려면 서비스로 등록합니다. `run.sh`가 실행 중이라면 먼저 `Ctrl+C`로 멈추세요.

```bash
./svc.sh install && ./svc.sh start
```

```bash
./svc.sh status
```

| 명령어 | 설명 |
|---|---|
| `./svc.sh install` | 서비스(LaunchAgent) 등록 |
| `./svc.sh start` | 서비스 시작 |
| `./svc.sh status` | 실행 상태 확인 |
| `./svc.sh stop` | 서비스 중지 |
| `./svc.sh uninstall` | 서비스 등록 해제 (러너 등록 정보는 남아 있음) |

서비스 설정 파일은 `~/Library/LaunchAgents/` 안에 `actions.runner.`로 시작하는 이름으로 만들어집니다.

### 4-3. 빌드 도구 경로 설정

러너는 내 터미널 설정(`~/.zshrc`)을 읽지 않습니다. 그래서 터미널에서는 되는 `flutter`, `pod`, `java`를 러너에서는 못 찾는 경우가 많습니다. 앱을 빌드할 계획이라면 설치 직후에 한 번 해 두세요.

**PATH 저장하기**: 필요한 명령어가 모두 잘 되는 터미널에서 실행합니다.

```bash
cd ~/actions-runners/REPO && echo "$PATH" > .path
```

**환경 변수 적기**: Android 빌드를 한다면 `JAVA_HOME`, `ANDROID_HOME`을 적습니다. 경로는 내 설치 위치에 맞게 바꾸세요.

```bash
echo 'JAVA_HOME=/Applications/Android Studio.app/Contents/jbr/Contents/Home' >> .env && echo "ANDROID_HOME=$HOME/Library/Android/sdk" >> .env
```

**러너 다시 시작하기**: `.path`, `.env`는 러너가 시작할 때 읽으므로 바꾼 뒤에는 꼭 다시 시작합니다.

```bash
./svc.sh stop && ./svc.sh start
```

플랫폼별로 필요한 도구는 [Android](docs/android.md#2-러너-mac-준비하기), [Flutter](docs/flutter.md#1-러너-mac-준비하기) 문서를 참고하세요.

### 4-4. 잠자기 방지 (선택)

컴퓨터가 잠자기에 들어가면 작업을 받지 못합니다. 계속 작업을 받아야 한다면 아래 중 하나를 쓰세요.

- **시스템 설정** → **배터리**(또는 **에너지**) → 전원 어댑터에 연결되어 있을 때 자동으로 잠자기 방지 옵션을 켭니다.
- 터미널에서 필요한 동안만 잠자기를 막습니다(`Ctrl+C`로 해제).

```bash
caffeinate -i
```

### 4-5. 러너 폴더 살펴보기

설치와 등록을 마치면 러너 폴더 안이 이렇게 채워집니다.

```
~/actions-runners/REPO/
├─ config.sh          러너 등록 / 등록 해제
├─ run.sh             러너를 직접 실행 (테스트용)
├─ svc.sh             서비스 등록·시작·중지 (상시 실행용)
├─ bin/, externals/   러너 프로그램 본체 (자동 업데이트 시 바뀜)
├─ .runner            등록 정보 (저장소 주소, 러너 이름 등)
├─ .credentials       GitHub 인증 정보 ← 절대 공유·커밋 금지
├─ .credentials_rsaparams
├─ .env               러너가 쓰는 환경 변수 (4-3에서 추가)
├─ .path              러너가 쓰는 PATH (4-3에서 저장)
├─ _diag/             러너 동작 로그
└─ _work/             빌드 작업 공간
    ├─ REPO/REPO/     checkout된 소스 코드 (작업 사이에 지워지지 않음)
    └─ _temp/         작업 중 임시 파일 (RUNNER_TEMP)
```

| 파일·폴더 | 언제 손대나요 |
|---|---|
| `.env` | 러너가 도구를 못 찾을 때 `JAVA_HOME`, `ANDROID_HOME` 같은 값을 적습니다. 적은 뒤 러너를 다시 시작합니다 |
| `.path` | `flutter`, `pod` 같은 명령어를 못 찾을 때 PATH를 고칩니다. 고친 뒤 러너를 다시 시작합니다 |
| `_diag/` | 러너가 이상할 때 로그를 확인합니다 |
| `_work/` | 디스크가 부족하거나 빌드가 꼬였을 때, 러너를 멈추고 지워도 됩니다. 다음 작업 때 다시 만들어집니다 |
| `.runner`, `.credentials*` | 직접 수정하지 않습니다. 다시 등록하려면 `config.sh remove` 후 새로 등록합니다 |

---

## 5. 워크플로에서 사용하기

저장소에 `.github/workflows/self-hosted-test.yml` 파일을 만듭니다.

```yaml
name: self-hosted test

on:
  workflow_dispatch:   # Actions 탭에서 직접 실행
  push:

jobs:
  hello:
    runs-on: [self-hosted, macOS, ARM64]   # Intel Mac이면 X64
    steps:
      - uses: actions/checkout@v7
      - name: Runner info
        run: |
          echo "Hello from self-hosted runner"
          uname -a
          sw_vers
```

- `runs-on`에 적은 라벨을 **모두 가진 러너**에서 작업이 실행됩니다.
- 러너가 하나뿐이라면 `runs-on: self-hosted`만 써도 됩니다.
- 등록할 때 직접 붙인 라벨(예: `flutter`)로도 고를 수 있습니다.

---

## 6. 동작 확인

1. 저장소 → **Settings** → **Actions** → **Runners**에서 러너가 **Idle**(초록색)로 보이는지 확인합니다.
2. 위 워크플로를 push하거나 **Actions** 탭 → 워크플로 선택 → **Run workflow**로 실행합니다.
3. 실행 중에는 러너 상태가 **Active**로 바뀌고, `./run.sh` 터미널에 `Running job: hello`가 표시됩니다.
4. 작업 로그의 `uname -a`, `sw_vers` 출력이 내 컴퓨터 정보와 같다면 성공입니다.

---

## 7. 보안 주의사항

self-hosted runner는 워크플로에 적힌 코드를 **내 컴퓨터에서, 내 사용자 권한으로 그대로 실행**합니다.

- **공개(public) 저장소에는 연결하지 마세요.** 누군가 저장소를 포크해서 PR을 보내면, 그 PR의 코드가 내 컴퓨터에서 실행될 수 있습니다. 공개 저장소에서 꼭 써야 한다면 **Settings → Actions → General**에서 외부 기여자의 워크플로 실행에 승인을 받도록 설정하세요.
- 러너는 내 파일, SSH 키, 브라우저 쿠키 같은 데이터에 접근할 수 있습니다. 가능하면 **러너 전용 macOS 사용자 계정**을 따로 만들어서 그 계정으로 설치하세요.
- 등록 토큰은 1시간 뒤 만료되지만, 공유하거나 문서·커밋에 남기지 마세요.
- 믿을 수 있는 액션만 쓰고, 가능하면 버전을 커밋 SHA로 고정하세요.

---

## 8. 문제 해결

| 증상 | 확인할 것 |
|---|---|
| Runners 목록에 **Offline**으로 보임 | `./run.sh`가 실행 중인지, 서비스라면 `./svc.sh status`로 상태를 확인합니다. 컴퓨터가 잠자기 상태인지, 인터넷이 연결되어 있는지도 확인합니다 |
| 작업이 계속 **Queued** 상태 | `runs-on` 라벨이 러너 라벨과 정확히 맞는지 확인합니다. 러너가 Offline이거나 다른 작업을 실행 중일 수도 있습니다 |
| `config.sh`에서 인증 오류 | 등록 토큰이 만료(1시간)되었을 수 있습니다. 새로 발급받으세요 |
| 서비스가 시작되지 않음 | 로그인한 사용자로 실행했는지 확인합니다. `sudo`로 `svc.sh`를 실행하지 마세요 |
| 워크플로에서 명령어를 찾을 수 없음 | 해당 도구가 설치되어 있는지 확인합니다. 서비스 실행 시 PATH가 다를 수 있으니 러너 폴더의 `.path` 파일을 확인하고, 수정했다면 서비스를 재시작합니다 |
| 러너가 목록에서 사라짐 | 14일 넘게 연결되지 않으면 자동으로 삭제됩니다. 다시 등록하세요 |

**로그 위치**

- 러너 동작 로그: `~/actions-runners/REPO/_diag/`
- 서비스 로그: `~/Library/Logs/actions.runner.*/`

---

## 9. 제거 방법

1. 서비스로 등록했다면 먼저 중지하고 해제합니다.

   ```bash
   cd ~/actions-runners/REPO
   ```

   ```bash
   ./svc.sh stop && ./svc.sh uninstall
   ```

2. 삭제용 토큰을 발급받습니다. 웹에서는 **Settings → Actions → Runners → 러너 선택 → Remove**를 누르면 명령어와 토큰이 나옵니다. CLI로는 아래처럼 발급합니다.

   ```bash
   gh api -X POST repos/OWNER/REPO/actions/runners/remove-token --jq .token
   ```

3. GitHub에서 러너 등록을 해제합니다.

   ```bash
   ./config.sh remove --token 삭제토큰
   ```

4. 러너 폴더를 지웁니다.

   ```bash
   cd ~ && rm -rf ~/actions-runners/REPO
   ```

---

## 10. 여러 저장소에서 쓰기

저장소에 등록한 러너는 **그 저장소의 작업만** 받습니다. 개인 계정에는 계정 전체에 러너를 등록하는 기능이 없어서, 여러 저장소에서 쓰려면 아래 둘 중 하나를 선택합니다.

**방법 1. 저장소마다 러너 폴더를 하나씩 추가하기**

`~/actions-runners/` 아래에 저장소 이름으로 폴더를 하나 더 만들고, [3-3](#3-3-러너-다운로드)부터 [4-3](#4-3-빌드-도구-경로-설정)까지를 그 폴더에서 반복합니다.

```bash
mkdir -p ~/actions-runners/app2 && cd ~/actions-runners/app2
```

```bash
./config.sh --url https://github.com/OWNER/app2 --token 등록토큰
```

- 러너 프로그램, 등록 정보, `.env`·`.path`, `_work`, 서비스가 **폴더마다 따로** 있습니다.
- 한 Mac에서 여러 러너를 동시에 실행할 수 있습니다.

**방법 2. Organization에 등록하기**

무료 Organization을 만들고 저장소를 그 안으로 옮기면, 조직에 러너 하나를 등록해서 여러 저장소가 같이 쓸 수 있습니다. 조직의 **Settings → Actions → Runners**에서 등록하며, 어떤 저장소가 쓸지도 정할 수 있습니다. 저장소가 많아질 예정이라면 이쪽이 관리하기 편합니다.

**러너끼리 공유되는 것**

러너 폴더는 따로여도, Mac에 설치된 것은 모든 러너가 같이 씁니다.

- 도구: Xcode, Flutter, JDK, Android SDK, CocoaPods
- 캐시: `~/.gradle`, `~/.pub-cache`, CocoaPods 캐시
- 프로비저닝 프로필 폴더

그래서 도구는 한 번만 설치하면 됩니다. 대신 여러 러너가 동시에 빌드하면 CPU와 메모리를 나눠 쓰므로 빌드가 느려질 수 있습니다.

---

## 참고 자료

- [GitHub Docs: Self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners)
- [GitHub Docs: Self-hosted runners reference (요구사항·네트워크)](https://docs.github.com/en/actions/reference/runners/self-hosted-runners)
- [actions/runner 릴리스](https://github.com/actions/runner/releases)

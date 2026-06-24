# Flutter CI/CD 환경 구축 — 실무 가이드

> GitLab Cloud Runner 기반 | Firebase App Distribution(dev) + Google Play Store(main) 자동 배포

---

## 전체 흐름

```
┌─────────────────────────────────────────────────────────────┐
│  dev 브랜치 push                                             │
│    → GitLab Pipeline (Cloud Runner)                         │
│      → 테스트 → 빌드(서명) → Firebase App Distribution 배포  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  main 브랜치 push (또는 태그)                                 │
│    → GitLab Pipeline (Cloud Runner)                         │
│      → 테스트 → 빌드(서명) → Google Play Store 배포           │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. 환경 정보

| 항목 | 내용 |
|---|---|
| CI/CD | GitLab CI (SaaS, 클라우드 Runner) |
| Runner | `saas-linux-medium-amd64` (GitLab 제공) |
| 베이스 이미지 | `ghcr.io/cirruslabs/flutter:3.22.2` |
| Flutter | 3.22.2 (Docker 이미지로 고정) |
| Java | 이미지 내장 (17) |
| 서명 | Keystore (CI Variables로 주입) |
| 내부 배포 | Firebase App Distribution |
| 스토어 배포 | Google Play Store (Fastlane) |

> **로컬 PC Runner 대비 장점**
> - PC 전원과 무관하게 24/7 빌드 가능
> - 팀원 환경에 의존하지 않는 격리된 Docker 환경
> - 경로 하드코딩 불필요 — 이미지 내부 경로 통일

---

## 2. 프로젝트 구조 (CI 관련 파일)

```
appname/
├── .gitlab-ci.yml            # 파이프라인 정의
├── fastlane/
│   ├── Fastfile              # 배포 자동화 스크립트
│   └── Appfile               # 앱 식별 정보
├── android/
│   ├── app/
│   │   └── build.gradle      # 서명 설정
│   └── gradle.properties
└── .gitignore                # *.jks, key.properties 반드시 포함
```

---

## 3. GitLab CI/CD Variables 설정

> `Settings → CI/CD → Variables → Add variable`

민감 정보는 반드시 **Masked + Protected** 적용.

| Key | 설명 | Masked | Protected |
|---|---|---|---|
| `KEYSTORE_BASE64` | `.jks` 파일을 base64 인코딩한 값 | ✅ | ✅ |
| `KEYSTORE_PASSWORD` | keystore 비밀번호 | ✅ | ✅ |
| `KEY_ALIAS` | key alias (예: `appname-key`) | ❌ | ❌ |
| `KEY_PASSWORD` | key 비밀번호 | ✅ | ✅ |
| `FIREBASE_APP_ID` | Firebase 앱 ID | ❌ | ❌ |
| `FIREBASE_TOKEN` | Firebase CLI 인증 토큰 | ✅ | ❌ |
| `PLAY_STORE_JSON_KEY` | Google Play API JSON 키 (전체 내용) | ✅ | ✅ |

### Keystore base64 인코딩 (로컬 최초 1회)

```bash
# macOS / Linux
base64 -i appname-release.jks | pbcopy   # 클립보드에 복사

# Windows PowerShell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("appname-release.jks")) | Set-Clipboard
```

---

## 4. android/app/build.gradle — 서명 설정

```groovy
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    compileSdk 36

    defaultConfig {
        applicationId "com....nas.appname"
        minSdk 21
        targetSdk 36
        versionCode 1
        versionName "1.0.0"
    }

    signingConfigs {
        release {
            keyAlias     keystoreProperties['keyAlias']
            keyPassword  keystoreProperties['keyPassword']
            storeFile    keystorePropertiesFile.exists()
                           ? file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'),
                          'proguard-rules.pro'
        }
    }
}
```

---

## 5. android/gradle.properties

```properties
org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=512m
android.useAndroidX=true
android.enableJetifier=false
```

---

## 6. Fastlane 설정

### fastlane/Appfile

```ruby
package_name("com....nas.appname")
json_key_file("fastlane/play-store-key.json")
```

### fastlane/Fastfile

```ruby
default_platform(:android)

platform :android do

  # ── 내부 테스터 배포 (dev 브랜치) ──────────────────────────
  lane :distribute_firebase do
    firebase_app_distribution(
      app:              ENV["FIREBASE_APP_ID"],
      firebase_cli_token: ENV["FIREBASE_TOKEN"],
      apk_path:         "build/app/outputs/flutter-apk/app-release.apk",
      groups:           "internal-testers",
      release_notes:    "#{ENV['CI_COMMIT_BRANCH']} — #{ENV['CI_COMMIT_SHORT_SHA']}\n#{ENV['CI_COMMIT_MESSAGE']}"
    )
  end

  # ── Google Play 내부 트랙 배포 (main 브랜치) ───────────────
  lane :deploy_play_store do
    upload_to_play_store(
      track:            "internal",
      aab:              "build/app/outputs/bundle/release/app-release.aab",
      skip_upload_apk:  true,
      release_status:   "draft"
    )
  end

end
```

> **AAB vs APK**
> - Play Store 업로드는 `.aab` (Android App Bundle) 형식을 사용합니다.
> - Firebase 배포는 `.apk` 형식을 사용합니다.
> - 두 형식을 모두 빌드하므로 각각 다른 커맨드를 사용합니다.

---

## 7. .gitlab-ci.yml (최종)

```yaml
image: ghcr.io/cirruslabs/flutter:3.22.2

stages:
  - test
  - build
  - distribute

# ── 공통: Keystore 복원 ────────────────────────────────────
.restore_keystore: &restore_keystore
  - echo "$KEYSTORE_BASE64" | base64 -d > android/appname-release.jks
  - |
    cat > android/key.properties <<EOF
    storePassword=${KEYSTORE_PASSWORD}
    keyPassword=${KEY_PASSWORD}
    keyAlias=${KEY_ALIAS}
    storeFile=../appname-release.jks
    EOF

# ── 공통: 빌드 후 민감 파일 제거 ──────────────────────────
.cleanup_keystore: &cleanup_keystore
  - rm -f android/appname-release.jks android/key.properties

# ══════════════════════════════════════════════════════════
# STAGE: test
# ══════════════════════════════════════════════════════════
unit_test:
  stage: test
  script:
    - flutter pub get
    - flutter test --coverage
  coverage: '/lines\s*:\s*([\d.]+)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura.xml
  rules:
    - if: '$CI_COMMIT_BRANCH == "dev" || $CI_COMMIT_BRANCH == "main"'

# ══════════════════════════════════════════════════════════
# STAGE: build
# ══════════════════════════════════════════════════════════

# ── dev: APK 빌드 (Firebase 배포용) ───────────────────────
build_apk:
  stage: build
  script:
    - *restore_keystore
    - flutter pub get
    - flutter build apk --release
    - *cleanup_keystore
  artifacts:
    name: "appname-apk-$CI_COMMIT_SHORT_SHA"
    paths:
      - build/app/outputs/flutter-apk/app-release.apk
    expire_in: 7 days
  rules:
    - if: '$CI_COMMIT_BRANCH == "dev"'
  needs:
    - unit_test

# ── main: AAB 빌드 (Play Store 업로드용) ──────────────────
build_aab:
  stage: build
  script:
    - *restore_keystore
    - flutter pub get
    - flutter build appbundle --release
    - *cleanup_keystore
  artifacts:
    name: "appname-aab-$CI_COMMIT_SHORT_SHA"
    paths:
      - build/app/outputs/bundle/release/app-release.aab
    expire_in: 30 days
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  needs:
    - unit_test

# ══════════════════════════════════════════════════════════
# STAGE: distribute
# ══════════════════════════════════════════════════════════

# ── dev: Firebase App Distribution 배포 ───────────────────
firebase_distribute:
  stage: distribute
  script:
    - gem install fastlane firebase_app_distribution --no-document
    - bundle exec fastlane distribute_firebase
  rules:
    - if: '$CI_COMMIT_BRANCH == "dev"'
  needs:
    - build_apk

# ── main: Google Play 내부 트랙 업로드 ────────────────────
play_store_upload:
  stage: distribute
  script:
    - gem install fastlane --no-document
    - echo "$PLAY_STORE_JSON_KEY" > fastlane/play-store-key.json
    - bundle exec fastlane deploy_play_store
    - rm -f fastlane/play-store-key.json
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  needs:
    - build_aab
```

---

## 8. .gitignore — 반드시 포함할 항목

```gitignore
# Android 서명 키 (절대 커밋 금지)
*.jks
*.keystore
android/key.properties

# Fastlane Play Store 인증키
fastlane/play-store-key.json
```

---

## 9. APK / AAB 다운로드

```
GitLab → CI/CD → Pipelines → 해당 Pipeline
  → build_apk (또는 build_aab) Job
    → Job artifacts → Download
```

---

## 10. 트러블슈팅

| 문제 | 원인 | 해결 |
|---|---|---|
| `base64: invalid input` | 줄바꿈 포함된 base64 | `base64 -w 0` 옵션으로 재인코딩 |
| `signing config not found` | `key.properties` 경로 오류 | `storeFile` 경로를 `../appname-release.jks`로 지정 |
| Firebase 배포 실패 | `FIREBASE_TOKEN` 만료 | `firebase login:ci`로 토큰 재발급 후 Variables 업데이트 |
| Play Store `Version code already exists` | 동일 versionCode 재업로드 | `build.gradle`의 `versionCode` 증가 |
| `compileSdk 36` 오류 | 플러그인 요구사항 | `build.gradle`에서 `compileSdk 36` 확인 |
| AAB 서명 안 됨 | `buildTypes.release.signingConfig` 누락 | `build.gradle` 4번 항목 재확인 |

---

## 11. 추후 계획

| 단계 | 내용 | 선행 조건 |
|---|---|---|
| iOS 빌드 | macOS Runner 추가, Xcode + Fastlane match 구성 | Apple 개발자 계정 |
| Play Store 자동 승인 | `release_status: "completed"` + `track: "production"` | 내부 검토 프로세스 |
| Slack 알림 | 빌드 성공/실패 시 Slack Webhook 연동 | Slack Incoming Webhook URL |
| 버전 자동 증가 | `versionCode`를 `CI_PIPELINE_IID`로 대체 | `build.gradle` 수정 |

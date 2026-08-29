# 📘 Модуль 9: CI/CD & DevOps для KMP

**В этом модуле вы освоите автоматизацию build, тестирования и деплоя KMP приложений: GitHub Actions, Gradle caching, multiplatform builds и release automation.**

**Цели модуля:**
1. Настроить CI/CD pipeline с GitHub Actions для KMP
2. Оптимизировать build performance с Gradle caching
3. Автоматизировать release process для Android и iOS
4. Настроить quality gates и automated testing

**Время выполнения:** ~30 часов.

---

## 1. GitHub Actions для KMP

### Basic CI Pipeline:

Создайте `.github/workflows/ci.yml`:

```yaml
name: KMP CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  GRADLE_OPTS: "-Dorg.gradle.daemon=false -Dorg.gradle.parallel=true"

jobs:
  # Lint and static analysis
  lint:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Cache Gradle
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-
      
      - name: Grant execute permission
        run: chmod +x gradlew
      
      - name: Run lint
        run: ./gradlew lint
        
      - name: Upload lint report
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: lint-report
          path: build/reports/lint

  # Unit tests for common code
  unit-tests:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Cache Gradle
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
      
      - name: Grant execute permission
        run: chmod +x gradlew
      
      - name: Run unit tests
        run: ./gradlew test
        
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results
          path: build/reports/tests

  # Android build and tests
  android-build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Setup Android SDK
        uses: android-actions/setup-android-sdk@v2
      
      - name: Cache Gradle
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
            .android
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
      
      - name: Decode keystore
        run: |
          echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 -d > android-app/keystore.jks
      
      - name: Create local.properties
        run: echo "sdk.dir=${{ steps.setup-android-sdk.outputs.sdk-dir }}" > local.properties
      
      - name: Grant execute permission
        run: chmod +x gradlew
      
      - name: Build Android debug
        run: ./gradlew :apps:android-app:assembleDebug
      
      - name: Run Android unit tests
        run: ./gradlew :apps:android-app:testDebugUnitTest
      
      - name: Run Android instrumented tests
        run: ./gradlew :apps:android-app:connectedAndroidTest
      
      - name: Upload APK
        uses: actions/upload-artifact@v3
        with:
          name: android-apk
          path: apps/android-app/build/outputs/apk/debug/*.apk

  # iOS build (requires macOS)
  ios-build:
    runs-on: macos-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Cache Gradle
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
      
      - name: Grant execute permission
        run: chmod +x gradlew
      
      - name: Build iOS simulator
        run: ./gradlew :shared:iosSimulatorArm64Binaries
      
      - name: Build iOS app with Xcode
        run: |
          xcodebuild \
            -project ios-app/SkillSync.xcodeproj \
            -scheme SkillSync \
            -sdk iphonesimulator \
            -configuration Debug \
            -derivedDataPath build/ios \
            CODE_SIGNING_ALLOWED=NO
      
      - name: Upload iOS app
        uses: actions/upload-artifact@v3
        with:
          name: ios-app
          path: build/ios/build/Products/Debug-iphonesimulator/SkillSync.app

  # Code coverage
  coverage:
    runs-on: ubuntu-latest
    needs: unit-tests
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Cache Gradle
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
      
      - name: Grant execute permission
        run: chmod +x gradlew
      
      - name: Generate coverage report
        run: ./gradlew createJacocoReport
      
      - name: Upload coverage report
        uses: actions/upload-artifact@v3
        with:
          name: coverage-report
          path: build/reports/jacoco
      
      - name: Upload to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: build/reports/jacoco/codeCoverage.xml
          fail_ci_if_error: false

  # Quality gate
  quality-gate:
    runs-on: ubuntu-latest
    needs: [lint, unit-tests, android-build, ios-build]
    
    steps:
      - name: Check all jobs passed
        run: |
          echo "All quality checks passed!"
          echo "✅ Lint"
          echo "✅ Unit tests"
          echo "✅ Android build"
          echo "✅ iOS build"

  # Merge job for PR status check
  ci-success:
    runs-on: ubuntu-latest
    needs: [quality-gate]
    
    steps:
      - name: CI Success
        run: echo "CI pipeline completed successfully!"
```

---

## 2. Gradle Caching & Performance

### Gradle Enterprise Configuration:

Создайте `gradle/init.gradle.kts`:

```kotlin
// Gradle Enterprise build scan
buildscript {
    repositories {
        maven("https://ge.gradle.com")
    }
    dependencies {
        classpath("com.gradle:enterprise-gradle-plugin:3.14.1")
    }
}

allprojects {
    apply(plugin = "com.gradle.enterprise")
    
    extensions.configure(com.gradle.enterprise.gradleplugin.GradleEnterpriseExtension::class) {
        buildScan {
            termsOfServiceUrl = "https://gradle.com/terms-of-service"
            termsOfServiceAgree = "yes"
            
            // Publish to Gradle Enterprise (optional)
            // publishAlways()
        }
    }
}

// Configure build cache
gradle.buildFinished {
    println("Build completed in ${gradle.buildResult.buildTime}ms")
}
```

### Local Build Cache:

Создайте `gradle/gradle-daemon-jvm.properties`:

```properties
# JVM options for Gradle daemon
-XX:MaxMetaspaceSize=512m
-Xmx4g
-XX:+HeapDumpOnOutOfMemoryError
```

### Build Cache Sharing:

Создайте `.github/workflows/build-cache.yml`:

```yaml
name: Build Cache

on:
  push:
    branches: [main]

jobs:
  publish-cache:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Configure Gradle build cache
        uses: gradle/gradle-build-action@v2
        with:
          cache-encryption-key: ${{ secrets.GRADLE_CACHE_ENCRYPTION_KEY }}
      
      - name: Generate cache key
        run: |
          echo "CACHE_KEY=${{ github.sha }}" >> $GITHUB_ENV
      
      - name: Cache Gradle
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
            .android
          key: gradle-cache-${{ github.sha }}
```

---

## 3. Release Automation

### Version Management:

Создайте `scripts/bump-version.sh`:

```bash
#!/bin/bash

# Usage: ./bump-version.sh [major|minor|patch]
set -e

VERSION_TYPE=${1:-patch}

# Read current version from gradle.properties
CURRENT_VERSION=$(grep "version=" shared/build.gradle.kts | cut -d'"' -f2)

# Parse version components
IFS='.' read -r MAJOR MINOR PATCH <<< "$CURRENT_VERSION"

# Bump version based on type
case $VERSION_TYPE in
  major)
    MAJOR=$((MAJOR + 1))
    MINOR=0
    PATCH=0
    ;;
  minor)
    MINOR=$((MINOR + 1))
    PATCH=0
    ;;
  patch)
    PATCH=$((PATCH + 1))
    ;;
esac

NEW_VERSION="$MAJOR.$MINOR.$PATCH"

echo "Bumping version from $CURRENT_VERSION to $NEW_VERSION"

# Update version in gradle.properties
sed -i "s/version=\"[^\"]*\"/version=\"$NEW_VERSION\"/" shared/build.gradle.kts

# Update version in Android
sed -i "s/versionName \"[^\"]*\"/versionName \"$NEW_VERSION\"/" apps/android-app/build.gradle.kts

# Update version in iOS (Info.plist)
/usr/libexec/PlistBuddy -c "Set :CFBundleShortVersionString $NEW_VERSION" ios-app/SkillSync/Info.plist

# Create git commit and tag
git add .
git commit -m "chore: bump version to $NEW_VERSION"
git tag "v$NEW_VERSION"

echo "✅ Version bumped to $NEW_VERSION"
```

### Release Pipeline:

Создайте `.github/workflows/release.yml`:

```yaml
name: Release Pipeline

on:
  release:
    types: [created]

jobs:
  # Build Android release
  android-release:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Setup Android SDK
        uses: android-actions/setup-android-sdk@v2
      
      - name: Decode keystore
        run: |
          echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 -d > android-app/keystore.jks
      
      - name: Create keystore.properties
        run: |
          cat > android-app/keystore.properties << EOF
          storeFile=keystore.jks
          storePassword=${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
          keyAlias=${{ secrets.ANDROID_KEY_ALIAS }}
          keyPassword=${{ secrets.ANDROID_KEY_PASSWORD }}
          EOF
      
      - name: Grant execute permission
        run: chmod +x gradlew
      
      - name: Build Android release
        run: ./gradlew :apps:android-app:assembleRelease
      
      - name: Sign Android APK
        run: ./gradlew :apps:android-app:signRelease
      
      - name: Upload APK to release
        uses: svenstaro/upload-release-action@v2
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          file: apps/android-app/build/outputs/apk/release/*.apk
          tag: ${{ github.ref }}
          file_glob: true
      
      - name: Upload AAB to release
        uses: svenstaro/upload-release-action@v2
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          file: apps/android-app/build/outputs/bundle/release/*.aab
          tag: ${{ github.ref }}
          file_glob: true

  # Build iOS release
  ios-release:
    runs-on: macos-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Grant execute permission
        run: chmod +x gradlew
      
      - name: Build iOS release
        run: |
          xcodebuild \
            -project ios-app/SkillSync.xcodeproj \
            -scheme SkillSync \
            -sdk iphoneos \
            -configuration Release \
            -derivedDataPath build/ios \
            CODE_SIGN_IDENTITY="${{ secrets.IOS_CODE_SIGNING_IDENTITY }}" \
            DEVELOPMENT_TEAM="${{ secrets.IOS_DEVELOPMENT_TEAM }}"
      
      - name: Archive iOS app
        run: |
          xcodebuild \
            -archivePath build/ios/SkillSync.xcarchive \
            -project ios-app/SkillSync.xcodeproj \
            -scheme SkillSync \
            -sdk iphoneos \
            -configuration Release \
            archive
      
      - name: Export IPA
        run: |
          xcodebuild \
            -exportArchive \
            -archivePath build/ios/SkillSync.xcarchive \
            -exportOptionsPlist export-options.plist \
            -exportPath build/ios/ipa
      
      - name: Upload IPA to release
        uses: svenstaro/upload-release-action@v2
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          file: build/ios/ipa/*.ipa
          tag: ${{ github.ref }}
          file_glob: true

  # Deploy to stores (optional)
  deploy-to-stores:
    runs-on: ubuntu-latest
    needs: [android-release, ios-release]
    
    steps:
      - name: Deploy to Google Play (internal testing)
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.GOOGLE_PLAY_SERVICE_ACCOUNT }}
          packageName: com.skillsync.app
          releaseFiles: apps/android-app/build/outputs/bundle/release/*.aab
          track: internal
      
      - name: Deploy to App Store Connect (TestFlight)
        uses: apple-actions/upload-to-app-store-connect@v1
        with:
          type: app
          bundle_id: com.skillsync.app
          build_path: build/ios/ipa/*.ipa
          api_key_id: ${{ secrets.APP_STORE_API_KEY_ID }}
          api_private_key: ${{ secrets.APP_STORE_API_PRIVATE_KEY }}
          issuer_id: ${{ secrets.APP_STORE_ISSUER_ID }}
```

---

## 4. Monitoring & Observability

### Crash Reporting Integration:

Создайте `shared/src/commonMain/kotlin/com/skillsync/monitoring/CrashReporter.kt`:

```kotlin
// commonMain - Crash reporting interface
interface CrashReporter {
    fun logError(
        error: Throwable,
        context: Map<String, String> = emptyMap()
    )
    
    fun logEvent(
        eventName: String,
        properties: Map<String, Any> = emptyMap()
    )
    
    fun setUser(
        userId: String,
        email: String? = null,
        username: String? = null
    )
}

// Android implementation with Firebase Crashlytics
class AndroidCrashReporter : CrashReporter {
    
    override fun logError(error: Throwable, context: Map<String, String>) {
        FirebaseCrashlytics.getInstance().recordError(error)
        
        context.forEach { (key, value) ->
            FirebaseCrashlytics.getInstance().setCustomKey(key, value)
        }
    }
    
    override fun logEvent(eventName: String, properties: Map<String, Any>) {
        val log = mutableMapOf<String, Any>()
        properties.forEach { (key, value) ->
            log[key] = value
        }
        
        FirebaseCrashlytics.getInstance().log("Event: $eventName")
    }
    
    override fun setUser(userId: String, email: String?, username: String?) {
        FirebaseCrashlytics.getInstance().setUserId(userId)
        
        email?.let { FirebaseCrashlytics.getInstance().setCustomKey("email", it) }
        username?.let { FirebaseCrashlytics.getInstance().setCustomKey("username", it) }
    }
}

// iOS implementation with Firebase Crashlytics
class IosCrashReporter : CrashReporter {
    
    override fun logError(error: Throwable, context: Map<String, String>) {
        FirebaseCrashlytics.crashlytics().recordError(error)
        
        context.forEach { (key, value) ->
            FirebaseCrashlytics.crashlytics().setCustomValue(value, key)
        }
    }
    
    override fun logEvent(eventName: String, properties: Map<String, Any>) {
        FirebaseCrashlytics.crashlytics().log("Event: $eventName")
    }
    
    override fun setUser(userId: String, email: String?, username: String?) {
        FirebaseCrashlytics.crashlytics().setUserId(userId)
        
        email?.let { FirebaseCrashlytics.crashlytics().setCustomValue(it, "email") }
        username?.let { FirebaseCrashlytics.crashlytics().setCustomValue(it, "username") }
    }
}

// Expect/actual for platform-specific implementation
expect fun createCrashReporter(): CrashReporter
```

---

## 📝 Домашнее задание (Модуль 9)

### Задача: Настройка CI/CD pipeline для SkillSync

**Требования:**
1. Создайте GitHub Actions workflow с lint, tests и builds для Android/iOS
2. Настройте Gradle caching для ускорения builds
3. Автоматизируйте release process с version bumping
4. Интегрируйте crash reporting (Firebase Crashlytics)

**Критерии сдачи:**
- ✅ CI pipeline проходит за < 15 минут
- ✅ Gradle cache снижает build time на >50%
- ✅ Release automation создает tagged releases
- ✅ Crash reporting настроен и отправляет данные

---

**Следующий модуль:** В Module_10 мы завершим Senior Track с mentorship и real-world project review.

Удачи! 🚀

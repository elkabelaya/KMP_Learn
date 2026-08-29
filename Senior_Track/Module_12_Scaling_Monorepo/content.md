# 📘 Модуль 7: Scaling & Monorepo Architecture

**В этом модуле вы освоите архитектуру больших KMP проектов: monorepo setup, module organization, dependency management и team collaboration.**

**Цели модуля:**
1. Настроить monorepo с Gradle для KMP проекта
2. Организовать модульную архитектуру (feature modules, core libraries)
3. Настроить dependency management и version catalog
4. Оптимизировать build performance для больших проектов

**Время выполнения:** ~35 часов.

---

## 1. Monorepo Structure для KMP

### Рекомендуемая структура проекта:

```
skillsync-monorepo/
├── apps/
│   ├── android-app/              # Android application
│   │   └── src/main/
│   ├── ios-app/                  # iOS application (Xcode project)
│   └── web-app/                  # Web application (optional)
│
├── shared/                       # KMP shared code
│   ├── core/                     # Core utilities (platform-agnostic)
│   │   └── src/commonMain/
│   ├── data/                     # Data layer (repositories, models)
│   │   └── src/commonMain/
│   ├── domain/                   # Business logic (use cases)
│   │   └── src/commonMain/
│   ├── presentation/             # UI layer (Compose)
│   │   └── src/commonMain/
│   ├── network/                  # Network layer (Ktor client)
│   │   └── src/commonMain/
│   ├── database/                 # Database layer (SQLDelight)
│   │   └── src/commonMain/
│   └── features/                 # Feature modules
│       ├── auth/                 # Authentication feature
│       │   └── src/commonMain/
│       ├── skills/               # Skills management feature
│       │   └── src/commonMain/
│       ├── profile/              # User profile feature
│       │   └── src/commonMain/
│       └── settings/             # Settings feature
│           └── src/commonMain/
│
├── build-logic/                  # Gradle plugins (convention plugins)
│   └── src/main/kotlin/
│       ├── skillsync.android.application.gradle.kts
│       ├── skillsync.android.library.gradle.kts
│       └── skillsync.kotlin.multiplatform.gradle.kts
│
├── gradle/
│   ├── libs.versions.toml        # Version catalog (BOM)
│   └── wrapper/
│
├── gradle.properties
├── settings.gradle.kts           # Monorepo settings
└── build.gradle.kts              # Root build file
```

### settings.gradle.kts:

```kotlin
pluginManagement {
    repositories {
        google()
        gradlePluginPortal()
        mavenCentral()
    }
    
    // Include build-logic for convention plugins
    includeBuild("build-logic")
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)
    
    repositories {
        google()
        mavenCentral()
        maven("https://maven.pkg.jetbrains.space/public/p/compose/dev")
    }
}

rootProject.name = "skillsync"

// Android app
include(":apps:android-app")

// Shared KMP modules
include(":shared:core")
include(":shared:data")
include(":shared:domain")
include(":shared:presentation")
include(":shared:network")
include(":shared:database")

// Feature modules
include(":shared:features:auth")
include(":shared:features:skills")
include(":shared:features:profile")
include(":shared:features:settings")

// Web app (optional)
// include(":apps:web-app")
```

### Root build.gradle.kts:

```kotlin
plugins {
    id("org.jetbrains.kotlin.multiplatform") version "1.9.21" apply false
    id("com.android.application") version "8.1.0" apply false
    id("com.android.library") version "8.1.0" apply false
}

allprojects {
    // Common configuration for all projects
    tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompile> {
        kotlinOptions {
            jvmTarget = "17"
            freeCompilerArgs += "-Xopt-in=kotlin.RequiresOptIn"
        }
    }
}

// Performance optimization for monorepo
gradle.allprojects {
    // Enable configuration cache
    tasks.withType<AbstractTask> {
        outputs.upToDateWhen { true } // For faster incremental builds
    }
}

// Parallel build configuration
gradle.startParameter.isParallelProjectExecution = true
```

---

## 2. Version Catalog (libs.versions.toml)

```toml
[versions]
kotlin = "1.9.21"
android-gradle-plugin = "8.1.0"
compose = "1.5.4"
ktor = "2.3.6"
sqldelight = "2.0.1"
koin = "3.5.0"
coroutines = "1.7.3"
serialization = "1.6.0"

[libraries]
# Kotlin
kotlin-stdlib = { module = "org.jetbrains.kotlin:kotlin-stdlib", version.ref = "kotlin" }
kotlin-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
kotlin-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "serialization" }

# Compose
compose-ui = { module = "org.jetbrains.compose.ui:ui", version.ref = "compose" }
compose-ui-tooling = { module = "org.jetbrains.compose.ui:ui-tooling", version.ref = "compose" }
compose-material3 = { module = "org.jetbrains.compose.material:material3", version.ref = "compose" }

# Ktor
ktor-client-core = { module = "io.ktor:ktor-client-core", version.ref = "ktor" }
ktor-client-content-negotiation = { module = "io.ktor:ktor-client-content-negotiation", version.ref = "ktor" }
ktor-serialization-kotlinx-json = { module = "io.ktor:ktor-serialization-kotlinx-json", version.ref = "ktor" }

# SQLDelight
sqldelight-runtime = { module = "app.cash.sqldelight:runtime", version.ref = "sqldelight" }
sqldelight-coroutines-extensions = { module = "app.cash.sqldelight:coroutines-extensions", version.ref = "sqldelight" }

# Koin
koin-core = { module = "io.insert-koin:koin-core", version.ref = "koin" }
koin-android = { module = "io.insert-koin:koin-android", version.ref = "koin" }

# Android
androidx-core-ktx = { module = "androidx.core:core-ktx", version = "1.12.0" }
androidx-activity-compose = { module = "androidx.activity:activity-compose", version = "1.8.0" }

[plugins]
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
android-application = { id = "com.android.application", version.ref = "android-gradle-plugin" }
android-library = { id = "com.android.library", version.ref = "android-gradle-plugin" }
compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
sqldelight = { id = "app.cash.sqldelight", version.ref = "sqldelight" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

---

## 3. Convention Plugins (Build Logic)

### shared-kmp.gradle.kts:

```kotlin
plugins {
    kotlin("multiplatform")
    kotlin("plugin.serialization")
}

kotlin {
    // Common configuration for all KMP modules
    
    // Source set hierarchy
    applyDefaultHierarchyTemplate {
        common {
            group("mobile") {
                withAndroidTarget()
                withIosTargets()
            }
        }
    }
    
    // Android target
    androidTarget {
        publishLibraryVariants("release", "debug")
        
        compilations.all {
            kotlinOptions {
                jvmTarget = "17"
            }
        }
    }
    
    // iOS targets
    iosX64()
    iosArm64()
    iosSimulatorArm64()
    
    sourceSets {
        val commonMain by getting {
            dependencies {
                // Common dependencies for all KMP modules
                implementation(libs.kotlin.coroutines.core)
                implementation(libs.kotlin.serialization.json)
            }
        }
        
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
            }
        }
    }
}

tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs += "-Xopt-in=kotlin.RequiresOptIn"
    }
}
```

### shared-kmp-compose.gradle.kts:

```kotlin
plugins {
    id("skillsync.kotlin.multiplatform")
    kotlin("plugin.compose")
}

kotlin {
    sourceSets {
        val commonMain by getting {
            dependencies {
                // Compose dependencies
                implementation(compose.ui)
                implementation(compose.material3)
            }
        }
        
        val androidMain by getting {
            dependencies {
                implementation(compose.uiTooling)
            }
        }
    }
}

android {
    namespace = "com.skillsync.${project.name}"
    
    defaultConfig {
        minSdk = 24
    }
}
```

---

## 4. Module Dependencies & Architecture

### Dependency Graph:

```
apps/android-app
    ├── shared:features:auth
    │   ├── shared:domain (use cases)
    │   ├── shared:data (repositories)
    │   └── shared:presentation (UI)
    ├── shared:features:skills
    │   ├── shared:domain
    │   ├── shared:data
    │   └── shared:presentation
    ├── shared:network (Ktor client)
    ├── shared:database (SQLDelight)
    └── shared:core (utilities, extensions)

shared:domain → NO dependencies on other modules
shared:data → depends on shared:domain, shared:network, shared:database
shared:presentation → depends on shared:domain
```

### Feature Module Example (auth/build.gradle.kts):

```kotlin
plugins {
    id("skillsync.kotlin.multiplatform.compose")
}

kotlin {
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation(project(":shared:domain"))
                implementation(project(":shared:data"))
                implementation(project(":shared:presentation"))
                
                // Koin for DI
                implementation(libs.koin.core)
            }
        }
        
        val androidMain by getting {
            dependencies {
                implementation(libs.koin.android)
                implementation(libs.androidx.activity.compose)
            }
        }
    }
}
```

---

## 5. Build Performance Optimization

### gradle.properties:

```properties
# Enable Gradle configuration cache
org.gradle.configuration-cache=true
org.gradle.configuration-cache.problems=fail

# Enable parallel builds
org.gradle.parallel=true

# Enable build cache
org.gradle.caching=true

# Increase heap size for large projects
org.gradle.jvmargs=-X4g -XX:MaxMetaspaceSize=512m

# Android-specific
android.useAndroidX=true
android.nonTransitiveRClass=true

# Kotlin compiler optimizations
kotlin.incremental=true
kotlin.incremental.java=true
```

### Build Metrics Task:

Создайте `build-logic/src/main/kotlin/tasks/BuildMetricsTask.kt`:

```kotlin
import org.gradle.api.DefaultTask
import org.gradle.api.tasks.TaskAction
import org.gradle.workers.WorkerDaemonService

abstract class BuildMetricsTask : DefaultTask() {
    
    @get:Input
    abstract val projectName: Property<String>
    
    @TaskAction
    fun collectMetrics() {
        println("📊 Build Metrics for ${projectName.get()}")
        println("----------------------------------------")
        
        // Collect build time, memory usage, etc.
        val buildTime = System.currentTimeMillis() - project.gradle.startParameter.projectBuildStart
        
        println("Build time: ${buildTime}ms")
        
        // Write to metrics file for CI/CD analysis
        val metricsFile = project.file("build/metrics.json")
        metricsFile.parentFile.mkdirs()
        
        metricsFile.writeText("""
            {
                "project": "${projectName.get()}",
                "buildTimeMs": $buildTime,
                "timestamp": ${System.currentTimeMillis()}
            }
        """)
    }
}
```

---

## 📝 Домашнее задание (Модуль 7)

### Задача: Рефакторинг SkillSync в monorepo архитектуру

**Требования:**
1. Перенесите код в monorepo структуру с feature modules
2. Настройте version catalog для всех зависимостей
3. Создайте convention plugins для KMP modules
4. Оптимизируйте build time (цель: < 3 минуты full clean build)

**Критерии сдачи:**
- ✅ Monorepo структура с четким разделением модулей
- ✅ Version catalog управляет всеми версиями зависимостей
- ✅ Convention plugins упрощают добавление новых модулей
- ✅ Build time оптимизирован с configuration cache

---

**Следующий модуль:** В Module_08 мы изучим advanced testing strategies для KMP.

Удачи! 🚀

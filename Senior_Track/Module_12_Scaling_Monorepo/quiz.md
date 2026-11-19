# 📝 Самопроверка: Модуль 12 — Scaling & Monorepo Architecture

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 10+ из 13 без подглядывания.

> ℹ️ Папка `Module_12_Scaling_Monorepo`, но внутри `content.md` этот модуль пронумерован как **Модуль 7**. Заголовок использует номер и название из самого модуля.

---

## Вопрос 1. Как устроена рекомендуемая структура монорепозитория?

- **A)** `apps/` (android-app, ios-app, web-app), `shared/` (core, data, domain, presentation, network, database, features/), `build-logic/`, `gradle/`, `settings.gradle.kts` и корневой `build.gradle.kts`
- **B)** Один единственный модуль `app/` со всеми исходниками — так проще собирать
- **C)** Отдельные Git-репозитории на каждую фичу, связанные через `git submodule`
- **D)** `src/main` и `src/ios` в корне, без Gradle-модулей

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Структура из модуля: приложения лежат в `apps/`, общий KMP-код — в `shared/` с разделением на `core/`, `data/`, `domain/`, `presentation/`, `network/`, `database/` и `features/` (auth, skills, profile, settings). `build-logic/` хранит convention-плагины, `gradle/libs.versions.toml` — версии зависимостей, `settings.gradle.kts` подключает все модули.

**Почему не другие:**
- B — это как раз монолит, от которого монорепо уходит: медленная сборка и неясная ответственность
- C — фичи — это Gradle-модули внутри одного репозитория, а не отдельные репозитории
- D — KMP использует source set'ы (`commonMain`, `androidMain`, `iosMain`), но они живут внутри модулей, а не в корне

</details>

---

## Вопрос 2. Зачем нужна папка `build-logic`?

- **A)** Чтобы хранить convention-плагины (`skillsync.android.application.gradle.kts`, `skillsync.android.library.gradle.kts`, `skillsync.kotlin.multiplatform.gradle.kts`) и переиспользовать одинаковую конфигурацию во всех модулях
- **B)** Чтобы хранить Kotlin-исходники бизнес-логики
- **C)** Чтобы хранить Apple-сертификаты для публикации в App Store
- **D)** Чтобы включить Detekt — больше ни для чего

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле перечислены три файла плагинов: `skillsync.android.application.gradle.kts`, `skillsync.android.library.gradle.kts`, `skillsync.kotlin.multiplatform.gradle.kts`. Они подключаются через `includeBuild("build-logic")` в `pluginManagement`, а feature-модуль затем пишет `id("skillsync.kotlin.multiplatform.compose")`. Это и есть централизация настроек: новый модуль получает target'ы, jvmTarget, зависимости «из коробки».

**Почему не другие:**
- B — для бизнес-логики есть `shared/*` и `features/*`
- C — сертификаты отношения к build-logic не имеют
- D — Detekt — это отдельный инструмент качества кода, не причина существования build-logic

</details>

---

## Вопрос 3. Что делают `includeBuild("build-logic")` и `RepositoriesMode.PREFER_SETTINGS`?

```kotlin
pluginManagement {
    repositories { google(); gradlePluginPortal(); mavenCentral() }
    includeBuild("build-logic")
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)
    repositories {
        google(); mavenCentral()
        maven("https://maven.pkg.jetbrains.space/public/p/compose/dev")
    }
}

rootProject.name = "skillsync"
include(":shared:features:auth")
```

- **A)** `includeBuild` подключает отдельный build с convention-плагинами; `PREFER_SETTINGS` задаёт репозитории централизованно в settings; `include` регистрирует модуль `:shared:features:auth`
- **B)** `includeBuild` копирует папку `build-logic` в каждый модуль; `PREFER_SETTINGS` отключает Maven Central
- **C)** `includeBuild` подключа Compose; `PREFER_SETTINGS` ускоряет компиляцию Kotlin
- **D)** `include` создаёт новую ветку Git для фичи auth

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`includeBuild("build-logic")` внутри `pluginManagement` — это composite build: плагины из `build-logic` становятся доступны по id без публикации в репозиторий. `RepositoriesMode.PREFER_SETTINGS` означает, что репозитории задаются в `dependencyResolutionManagement` и имеют приоритет над объявлениями в самих проектах. Строки `include(":shared:core")`, `include(":shared:features:auth")` и т.д. регистрируют модули в Gradle по их путям.

**Почему не другие:**
- B — ничего не копируется; Maven Central, наоборот, присутствует в списке
- C — Compose подключается зависимостью `org.jetbrains.compose...`, а `PREFER_SETTINGS` не влияет на скорость компиляции
- D — `include()` — метод Gradle для регистрации проектов, а не Git-команда

</details>

---

## Вопрос 4. Что такое `gradle/libs.versions.toml` и какие секции в нём есть?

- **A)** Version catalog: `[versions]` (версии), `[libraries]` (координаты с `version.ref`), `[plugins]` (id плагинов) — единое место управления версиями
- **B)** Файл настроек Kotlin-компилятора: там хранятся `jvmTarget` и флаги `-Xopt-in`
- **C)** Gradle wrapper: хранит URL и контрольную сумму дистрибутива Gradle
- **D)** Отчёт Detekt со списком нарушений

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле показан каталог с `[versions]` (`kotlin = "1.9.21"`, `ktor = "2.3.6"`, `sqldelight = "2.0.1"`, `koin = "3.5.0"` и др.), `[libraries]` (`kotlin-coroutines-core = { module = "...", version.ref = "coroutines" }`) и `[plugins]` (`kotlin-multiplatform`, `android-application`, `compose-compiler`, `sqldelight`, `kotlin-serialization`). В `build.gradle.kts` это используется как `libs.kotlin.coroutines.core` — версия меняется в одном месте.

**Почему не другие:**
- B — компилятор настраивается в `kotlinOptions`/`freeCompilerArgs`, а не в TOML-каталоге
- C — wrapper лежит в `gradle/wrapper/`
- D — отчёты Detekt создаются в `build/reports/`

</details>

---

## Вопрос 5. Как эта запись из version catalog будет называться в `build.gradle.kts`?

```toml
[versions]
coroutines = "1.7.3"

[libraries]
kotlin-coroutines-core = {
    module = "org.jetbrains.kotlinx:kotlinx-coroutines-core",
    version.ref = "coroutines"
}
```

- **A)** `implementation(libs.kotlin.coroutines.core)` — версия `1.7.3`
- **B)** `implementation(libs.coroutines)` — версия `1.7.3`
- **C)** `implementation(libs.kotlin.coroutines.core)` — версия возьмётся из `[versions] kotlin`, то есть `1.9.21`
- **D)** `implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core")` без версии — Gradle resolved её сам

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Имя алиаса `kotlin-coroutines-core` превращается в accessor `libs.kotlin.coroutines.core` (дефисы → точки). Версия берётся из `version.ref = "coroutines"`, а значит это `1.7.3`. В модуле именно так и написано в convention-плагине: `implementation(libs.kotlin.coroutines.core)`.

**Почему не другие:**
- B — shortened алиас `libs.coroutines` не соответствует ключу `kotlin-coroutines-core`
- C — `version.ref` явно указывает на `coroutines`, а не на `kotlin`
- D — без явной версии или `version.ref` каталог не сработает; в конфиге версия обязательна

</details>

---

## Вопрос 6. Что настраивает `applyDefaultHierarchyTemplate` в convention-плагине `shared-kmp`?

```kotlin
applyDefaultHierarchyTemplate {
    common {
        group("mobile") {
            withAndroidTarget()
            withIosTargets()
        }
    }
}

androidTarget { publishLibraryVariants("release", "debug") }
iosX64(); iosArm64(); iosSimulatorArm64()
```

- **A)** Создаёт общий source set `mobile` для Android и iOS внутри `common`, плюс объявляет три iOS-таргета и Android-таргет с jvmTarget 17
- **B)** Удаляет `commonMain` — каждый таргет живёт отдельно
- **C)** Объединяет Android и Desktop в один source set `desktop`
- **D)** Отключает публикацию вариантов `release` и `debug`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`group("mobile") { withAndroidTarget(); withIosTargets() }` создаёт промежуточный source set `mobileMain`, общий для Android и трёх iOS-таргетов (внутри `common`). Это позволяет делить код между мобильными платформами, но не тянуть его в Desktop/Web. Далее объявлены `androidTarget` с публикацией `release`/`debug` и jvmTarget 17, а также `iosX64`, `iosArm64`, `iosSimulatorArm64`.

**Почему не другие:**
- B — `commonMain` сохраняется и содержит общие зависимости (coroutines, serialization)
- C — в группе `mobile` — Android и iOS, Desktop здесь не упоминается
- D — наоборот: `publishLibraryVariants("release", "debug")` включает публикацию обоих вариантов

</details>

---

## Вопрос 7. Чем отличается плагин `skillsync.kotlin.multiplatform.compose` от базового `skillsync.kotlin.multiplatform`?

- **A)** Он применяет базовый плагин + `kotlin("plugin.compose")`, добавляет `compose.ui`/`compose.material3` в `commonMain`, `compose.uiTooling` в `androidMain`, задаёт `namespace = "com.skillsync.${project.name}"` и `minSdk = 24`
- **B)** Он полностью заменяет KMP-плагин и работает только на Android
- **C)** Он добавляет SQLDelight и Ktor, но не Compose
- **D)** Он идентичен базовому, просто другое имя

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В коде плагина:

```kotlin
plugins {
    id("skillsync.kotlin.multiplatform")
    kotlin("plugin.compose")
}
```

Далее `implementation(compose.ui)`, `implementation(compose.material3)` в `commonMain`; `implementation(compose.uiTooling)` в `androidMain`; блок `android { namespace = "com.skillsync.${project.name}"; defaultConfig { minSdk = 24 } }`. Feature-модуль `auth` подключает именно этот плагин.

**Почему не другие:**
- B — плагин расширяет, а не заменяет базовый; Compose Multiplatform остаётся кроссплатформенным
- C — SQLDelight и Ktor — отдельные зависимости каталога (`sqldelight-runtime`, `ktor-client-core`)
- D — отличия есть, и они перечислены выше

</details>

---

## Вопрос 8. У какого модуля в графе зависимостей **нет** зависимостей от других модулей?

- **A)** `shared:core`
- **B)** `shared:domain`
- **C)** `shared:data`
- **D)** `shared:features:auth`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В графе прямо сказано: `shared:domain → NO dependencies on other modules`. Остальное: `shared:data → domain, network, database`; `shared:presentation → domain`; `apps/android-app → features, network, database, core`; `features:auth → domain, data, presentation`. Domain — ядро, не зависящее ни от чего, поэтому его легко тестировать и переиспользовать.

**Почему не другие:**
- A — `shared:core` в графе – utilities, extensions, от него зависят другие модули (но это не ядро с запретом зависимостей по условию графа)
- C — data зависит от domain, network и database
- D — auth зависит от domain, data и presentation (см. `build.gradle.kts` фичи)

</details>

---

## Вопрос 9. Что не так с этим фрагментом `build.gradle.kts` фичи auth?

```kotlin
val commonMain by getting {
    dependencies {
        implementation(project(":shared:domain"))
        implementation(project(":shared:data"))
        implementation(project(":shared:presentation"))
        implementation(libs.koin.android)   // ← ?
    }
}
```

- **A)** Ошибки нет: Koin для Android можно подключать в `commonMain`
- **B)** `libs.koin.android` — Android-зависимость: её нужно перенести в `androidMain`, а в `commonMain` оставить `libs.koin.core`
- **C)** Нужно удалить все `project(...)` зависимости, фича должна быть полностью автономной
- **D)** Вместо `libs.koin.core` нужно подключить `libs.androidx.activity.compose`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В эталонном файле фичи в `commonMain` стоит `implementation(libs.koin.core)`, а `libs.koin.android` — в `androidMain` рядом с `libs.androidx.activity.compose`. Android-специфичная зависимость в common-коде не скомпилируется для iOS. Это вопрос-ловушка на границы source set'ов.

**Почему не другие:**
- A — это и есть ошибка: `commonMain` компилируется для всех target'ов
- C — фича намеренно зависит от `domain`, `data`, `presentation`
- D — `androidx.activity.compose` — это Android-зависимость и живёт в `androidMain`, заменять koin.core им нельзя

</details>

---

## Вопрос 10. Какие флаги optimize сборки включены в `gradle.properties` модуля?

- **A)** `org.gradle.configuration-cache=true` (и `problems=fail`), `org.gradle.parallel=true`, `org.gradle.caching=true`, `org.gradle.jvmargs=-X4g`, `android.nonTransitiveRClass=true`, `kotlin.incremental=true`
- **B)** Только `org.gradle.parallel=true`, остальное не включается
- **C)** `org.gradle.configuration-cache=false`, зато `kotlin.incremental=false`
- **D)** Только флаги Detekt и SwiftLint

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Полный список из модуля: configuration cache (`true` + `problems=fail`), parallel, build cache, `-X4g -XX:MaxMetaspaceSize=512m`, `android.useAndroidX=true`, `android.nonTransitiveRClass=true`, `kotlin.incremental=true` и `kotlin.incremental.java=true`. Вместе они ускоряют сборку большого монорепозитория.

**Почему не другие:**
- B — оптимизаций несколько, они перечислены все
- C — всё с точностью до наоборот: кэш и инкремент включены
- D — линтеры настраиваются в CI и `detekt-config.yml`, а не в `gradle.properties`

</details>

---

## Вопрос 11. Что делает `BuildMetricsTask` из `build-logic/src/main/kotlin/tasks/BuildMetricsTask.kt`?

```kotlin
abstract class BuildMetricsTask : DefaultTask() {
    @get:Input
    abstract val projectName: Property<String>

    @TaskAction
    fun collectMetrics() {
        val buildTime = System.currentTimeMillis() - project.gradle.startParameter.projectBuildStart
        val metricsFile = project.file("build/metrics.json")
        metricsFile.parentFile.mkdirs()
        metricsFile.writeText("""{ "project": "...", "buildTimeMs": $buildTime, ... }""")
    }
}
```

- **A)** Собирает метрики сборки (имя проекта, время, timestamp) и записывает их в `build/metrics.json` для анализа в CI/CD
- **B)** Запускает юнит-тесты и публикует отчёт в HTML
- **C)** Проверяет Kotlin-код на соответствие стилю Detekt
- **D)** Генерирует Gradle wrapper

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`@TaskAction` помечает метод как точку входа задачи, `@get:Input` объявляет входное свойство `projectName`. Внутри считается `buildTime`, создаётся папка `build/`, и в `metrics.json` пишутся `project`, `buildTimeMs` и `timestamp`. Комментарий в модуле: «Write to metrics file for CI/CD analysis».

**Почему не другие:**
- B — тесты — это `./gradlew :shared:test`, а не метрики
- C — для стиля есть `detekt` и `ktlintCheck`
- D — wrapper создаётся командой `gradle wrapper`, не этой задачей

</details>

---

## Вопрос 12. Какова цель по времени сборки и критерии сдачи домашнего задания?

- **A)** Full clean build < 3 минут; чёткая монорепо-структура; version catalog управляет всеми версиями; convention-плагины упрощают добавление модулей; configuration cache включён
- **B)** Full clean build < 30 секунд при любом размере проекта; можно обойтись без version catalog
- **C)** Время сборки не измеряется, главное — количество модулей
- **D)** Достаточно включить parallel — остальное необязательно

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Требование домашнего задания: «Оптимизируйте build time (цель: < 3 минуты full clean build)». Критерии сдачи: ✅ монорепо с чётким разделением модулей; ✅ version catalog управляет всеми версиями; ✅ convention plugins упрощают добавление новых модулей; ✅ build time оптимизирован с configuration cache.

**Почему не другие:**
- B — порог 3 минуты, а не 30 секунд; version catalog обязателен
- C — время сборки — один из ключевых критериев
- D — одного parallel мало: нужны configuration cache, build cache и инкремент

</details>

---

## Вопрос 13. ❗Ловушка. Что не так с этим «ускорением» в корневом `build.gradle.kts`?

```kotlin
gradle.allprojects {
    // Enable configuration cache
    tasks.withType<AbstractTask> {
        outputs.upToDateWhen { true } // For faster incremental builds
    }
}
gradle.startParameter.isParallelProjectExecution = true
```

- **A)** Всё верно: так правильно включается configuration cache
- **B)** `outputs.upToDateWhen { true }` принудительно помечает **все** задачи актуальными — Gradle будет пропускать компиляцию и тесты, сборка станет некорректной
- **C)** Нужно заменить `true` на `false` у всех задач — тогда сборка станет быстрее
- **D)** Проблема в `isParallelProjectExecution`: параллельную сборку включать нельзя

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Это вопрос-ловушка: комментарий в модуле обещает «быстрые инкрементальные сборки», но `upToDateWhen { true }` заставляет Gradle считать все задачи актуальными без проверки входов. Изменённый Kotlin-код не перекомпилируется, тесты не запускаются — сборка «быстрая» лишь потому, что не делает работы. Настоящая оптимизация — из `gradle.properties`: `org.gradle.configuration-cache=true`, `org.gradle.caching=true`, `kotlin.incremental=true`, `org.gradle.parallel=true`.

**Почему не другие:**
- A — configuration cache включается в `gradle.properties`, а не через `upToDateWhen`
- C — `false` заставит перевыполнять всё каждый раз, что медленно и тоже не является «инкрементом»
- D — параллельное выполнение_project'ов как раз рекомендуется в модуле (`org.gradle.parallel=true`)

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 11–13 | 🎉 Отлично: монорепо, version catalog и build-оптимизации — твои |
| 8–10 | 📖 Перечитай секции про `settings.gradle.kts`, version catalog и граф зависимостей |
| 4–7 | ⚠️ Пройди модуль ещё раз: структура `apps/shared/build-logic` и convention plugins |
| < 4 | 🆘 Начни с структуры проекта и `libs.versions.toml`, потом вернись к квизу |

---

> 🔜 Дальше в Senior Track — продвинутое тестирование, сложные темы, CI/CD и менторство. Применяй знания о монорепо на реальных проектах! 🚀

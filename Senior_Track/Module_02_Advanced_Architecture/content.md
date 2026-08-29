# 📘 Модуль 1: Продвинутая архитектура и модульность

**Добро пожаловать на Senior Track!**  
В этом модуле мы перейдем от базовой архитектуры к enterprise-уровню. Вы научитесь проектировать сложные системы, разбивать их на модули и управлять зависимостями в больших проектах.

**Цели модуля:**
1. Освоить Domain-Driven Design (DDD) в контексте KMP
2. Научиться разбивать проект на feature-модули
3. Освоить Gradle Version Catalogs и Composite Builds
4. Создать собственный Gradle плагин для KMP

**Время выполнения:** ~30–40 часов (6 недель).

---

## 1. Введение: Почему модульность важна?

В базовом курсе мы создали монолитный проект с разделением на слои (UI, Domain, Data). Это хорошо для небольших приложений. Но что делать, когда:
- Приложение растет до 100k+ строк кода?
- Над проектом работает команда из 10+ разработчиков?
- Нужно переиспользовать код в нескольких приложениях?

**Проблемы монолита:**
- 🐌 Медленная сборка (30+ минут)
- 🔗 Циклические зависимости между модулями
- 👥 Конфликты при слиянии кода (merge conflicts)
- 📦 Большой размер APK (все зависимости в одном бандле)

**Решение: Модульная архитектура**
Разбиваем проект на независимые модули, каждый из которых:
- Имеет четкую ответственность (Single Responsibility Principle)
- Собирается независимо
- Может переиспользоваться в других проектах

---

## 2. Теория: Domain-Driven Design (DDD) в KMP

### Что такое DDD?
DDD — это подход к проектированию сложных систем, где код отражает бизнес-домен.

### Ключевые концепции:

#### 🎯 Bounded Context (Ограниченный контекст)
Границы, внутри которых бизнес-термины имеют однозначное значение.

**Пример в SkillSync:**
```
Контекст "Обучение":
- Сессия (Session) = встреча ментора и ученика
- Навык (Skill) = то, чему можно научить

Контекст "Платежи":
- Сессия (Session) = платная транзакция
- Навык (Skill) = тарифный план
```

Один и тот же термин может иметь разное значение в разных контекстах!

#### 📦 Aggregate (Агрегат)
Группа сущностей, которые всегда изменяются вместе.

**Пример:**
```kotlin
// Aggregate Root
class User(
    val id: UserId,
    var name: String,
    private val skills: MutableList<UserSkill> = mutableListOf()
) {
    // Метод агрегата (не просто setter!)
    fun addSkill(skill: Skill, level: SkillLevel) {
        // Бизнес-логика: нельзя добавить дубликат
        require(!skills.any { it.skillId == skill.id }) {
            "Skill already exists"
        }
        skills.add(UserSkill(skill, level))
    }
}
```

#### 🎪 Domain Events (События домена)
Что-то важное произошло в системе.

```kotlin
sealed class DomainEvent {
    data class SkillAdded(val userId: UserId, val skillId: SkillId) : DomainEvent()
    data class SessionScheduled(val sessionId: SessionId, val date: Instant) : DomainEvent()
}

class SkillRepository {
    fun addSkill(user: User, skill: Skill) {
        // ... сохраняем в БД
        // Публикуем событие
        eventBus.publish(SkillAdded(user.id, skill.id))
    }
}
```

---

## 3. Теория: Модульная структура проекта

### 📂 Рекомендуемая структура для SkillSync

```
SkillSync/
├── apps/
│   ├── skillsync-android/          # Android приложение
│   ├── skillsync-ios/              # iOS приложение (Xcode)
│   ├── skillsync-desktop/          # Desktop приложение
│   └── skillsync-web/              # Web приложение (Compose Web)
├── shared/
│   ├── core/
│   │   ├── core-common/            # Общие утилиты (Result, Either, Extensions)
│   │   ├── core-ui/                # Общие UI компоненты (кнопки, инпуты)
│   │   ├── core-data/              # Общие репозитории и модели
│   │   └── core-network/           # Ktor Client конфигурация
│   ├── feature/
│   │   ├── feature-auth/           # Аутентификация (вход, регистрация)
│   │   ├── feature-skills/         # Каталог навыков
│   │   ├── feature-chat/           # Чат между пользователями
│   │   ├── feature-sessions/       # Календарь сессий
│   │   ├── feature-profile/        # Профиль пользователя
│   │   └── feature-settings/       # Настройки приложения
│   └── domain/
│       ├── domain-models/          # Entity классы (User, Skill, Session)
│       └── domain-usecases/        # UseCase интерфейсы
├── build.gradle.kts                # Root build файл
└── settings.gradle.kts             # Настройка модулей
```

### 🔗 Зависимости между модулями (DAG - Directed Acyclic Graph)

```
apps/* → feature-* → core-* → domain-*
                    ↓
              (нет обратных зависимостей!)
```

**Правило:** Модуль нижнего уровня НЕ может зависеть от модуля верхнего уровня.

---

## 4. Практика: Создание модульной структуры SkillSync

### Шаг 1: Настройка root build файла

Создайте `build.gradle.kts` в корне проекта:

```kotlin
plugins {
    kotlin("multiplatform") version "2.0.0" apply false
    kotlin("plugin.serialization") version "2.0.0" apply false
    id("com.android.application") version "8.1.0" apply false
    id("com.android.library") version "8.1.0" apply false
}

// Включаем все модули
includeBuild("shared/core/core-common")
includeBuild("shared/domain/domain-models")
includeBuild("shared/feature/feature-auth")
// ... остальные модули
```

### Шаг 2: Настройка settings.gradle.kts

```kotlin
rootProject.name = "SkillSync"

// Включаем приложения
include(":apps:skillsync-android")
include(":apps:skillsync-ios")
include(":apps:skillsync-desktop")

// Включаем core модули
include(":shared:core:core-common")
include(":shared:core:core-ui")
include(":shared:core:core-data")

// Включаем feature модули
include(":shared:feature:feature-auth")
include(":shared:feature:feature-skills")
include(":shared:feature:feature-chat")

// Включаем domain модули
include(":shared:domain:domain-models")
include(":shared:domain:domain-usecases")
```

### Шаг 3: Создание core-common модуля

Создайте `shared/core/core-common/build.gradle.kts`:

```kotlin
plugins {
    kotlin("multiplatform")
}

kotlin {
    androidTarget()
    iosX64()
    iosArm64()
    iosSimulatorArm64()
    jvm() // Для Desktop
    
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
            }
        }
    }
}
```

### Шаг 4: Реализация Result типа в commonMain

Создайте `shared/core/core-common/src/commonMain/kotlin/result/Result.kt`:

```kotlin
package result

// Кастомный Result для обработки ошибок
sealed class Result<out T, out E> {
    data class Success<out T>(val data: T) : Result<T, Nothing>()
    data class Failure<out E>(val error: E) : Result<Nothing, E>
    
    inline fun <R> map(transform: (T) -> R): Result<R, E> =
        when (this) {
            is Success -> Success(transform(data))
            is Failure -> this as Result<R, E>
        }
    
    inline fun <R> mapError(transform: (E) -> R): Result<T, R> =
        when (this) {
            is Success -> this as Result<T, R>
            is Failure -> Failure(transform(error))
        }
    
    inline fun <R> flatMap(transform: (T) -> Result<R, E>): Result<R, E> =
        when (this) {
            is Success -> transform(data)
            is Failure -> this as Result<R, E>
        }
}

// Extension функции
fun <T> T.asResult(): Result<T, Nothing> = Result.Success(this)

inline fun <T, E> result(block: () -> T): Result<T, E> =
    try {
        Result.Success(block())
    } catch (e: Exception) {
        @Suppress("UNCHECKED_CAST")
        Result.Failure(e as E)
    }
```

---

## 5. Практика: Version Catalogs для управления зависимостями

### Что такое Version Catalogs?
Единое место для управления версиями всех зависимостей в проекте.

### Шаг 1: Создание libs.versions.toml

Создайте `gradle/libs.versions.toml` в корне:

```toml
[versions]
kotlin = "2.0.0"
compose = "1.6.0"
koin = "3.5.0"
sqldelight = "2.0.0"
ktor = "2.3.0"

[libraries]
kotlinx-coroutines-core = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-core", version.ref = "kotlin" }
koin-core = { group = "io.insert-koin", name = "koin-core", version.ref = "koin" }
sqldelight-runtime = { group = "app.cash.sqldelight", name = "runtime", version.ref = "sqldelight" }
ktor-client-core = { group = "io.ktor", name = "ktor-client-core", version.ref = "ktor" }

[plugins]
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
koin = { id = "io.insert-koin.plugin", version.ref = "koin" }
```

### Шаг 2: Использование в модулях

В `shared/core/core-common/build.gradle.kts`:

```kotlin
dependencies {
    implementation(libs.kotlinx.coroutines.core)
}
```

**Преимущества:**
- ✅ Все версии в одном месте
- ✅ Легко обновлять (изменил в toml — обновились везде)
- ✅ Нет дублирования версий

---

## 6. Практика: Создание Gradle плагина для KMP модулей

### Зачем свой плагин?
Чтобы не копировать одинаковый код конфигурации в каждый модуль.

### Шаг 1: Создание плагина

Создайте `gradle/plugins/build.gradle.kts`:

```kotlin
plugins {
    `java-gradle-plugin`
}

dependencies {
    implementation(libs.kotlin.gradle.plugin)
}
```

### Шаг 2: Реализация плагина

Создайте `gradle/plugins/src/main/kotlin/SkillSyncKmpPlugin.kt`:

```kotlin
class SkillSyncKmpPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        project.plugins.apply("org.jetbrains.kotlin.multiplatform")
        
        project.extensions.configure<KotlinMultiplatformExtension> {
            androidTarget()
            iosX64()
            iosArm64()
            iosSimulatorArm64()
            jvm()
            
            sourceSets {
                getByName("commonMain") {
                    dependencies {
                        implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
                    }
                }
            }
        }
    }
}
```

### Шаг 3: Использование плагина

В любом новом модуле просто добавьте:

```kotlin
plugins {
    id("skill-sync-kmp")
}
```

---

## 📝 Домашнее задание (Модуль 1)

### Задача: Создать модульную архитектуру для SkillSync

**Требования:**

1. **Создайте структуру проекта:**
   - 3 core-модуля (core-common, core-ui, core-data)
   - 3 feature-модуля (feature-auth, feature-skills, feature-profile)
   - 2 domain-модуля (domain-models, domain-usecases)

2. **Настройте зависимости:**
   - feature-* зависит от core-* и domain-*
   - core-data зависит от domain-models
   - Нет циклических зависимостей

3. **Реализуйте DDD концепции:**
   - Создайте 2 Bounded Context (например, "Обучение" и "Платежи")
   - Создайте 1 Aggregate (например, User с навыками)
   - Реализуйте Domain Events для одного события

4. **Настройте Version Catalog:**
   - Перенесите все зависимости в libs.versions.toml
   - Используйте alias'ы во всех модулях

5. **Создайте Gradle плагин:**
   - Автоматически настраивает KMP таргеты
   - Добавляет общие зависимости

**Критерии сдачи:**
- ✅ Все модули собираются независимо (`./gradlew :shared:feature:feature-auth:assemble`)
- ✅ Нет циклических зависимостей (проверьте через `./gradlew dependencies`)
- ✅ Version Catalog управляет всеми версиями
- ✅ Gradle плагин упрощает создание новых модулей
- ✅ DDD концепции реализованы корректно

**Бонусные задания:**
- Настройте Gradle Configuration Cache для ускорения сборки
- Создайте плагин для автоматической генерации модулей из шаблона

---

## 💡 Советы по выполнению

1. **Начните с малого:** Сначала создайте 2-3 модуля, убедитесь что они работают вместе.
2. **Тестируйте часто:** После каждого добавления модуля делайте `./gradlew build`.
3. **Используйте граф зависимостей:** `./gradlew :shared:feature:feature-auth:dependencies --configuration commonMainCompileClasspath`
4. **Не усложняйте:** Если модуль можно объединить — объедините. Модульность не самоцель.

---

**Следующий модуль:** В Module_02_Advanced_Compose мы изучим кастомные Layout'ы, анимации и оптимизацию производительности Compose UI.

Удачи! 🚀

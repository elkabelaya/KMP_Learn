# 📝 Самопроверка: Модуль 2 — Продвинутая архитектура и модульность

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 9+ из 12 без подглядывания.

---

## Вопрос 1. Какая проблема НЕ отнесена в модуле к типичным проблемам монолита?

- **A)** Медленная сборка (30+ минут)
- **B)** Циклические зависимости между модулями
- **C)** Merge-конфликты при работе большой команды
- **D)** Слишком маленький размер APK (всего 2 МБ)

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: D**

В модуле перечислены: 🐌 медленная сборка, 🔗 циклические зависимости, 👥 merge-конфликты и 📦 **большой** размер APK (все зависимости в одном бандле). Маленький APK — не проблема, а наоборот, желаемый результат модульности.

**Почему не другие:**
- A, B, C — прямо названы в списке проблем монолита

</details>

---

## Вопрос 2. `Session` в контексте «Обучение» — встреча ментора и ученика, а в контексте «Платежи» — платная транзакция. Что это иллюстрирует?

- **A)** Один и тот же термин может иметь разное значение в разных Bounded Context's
- **B)** Нужно объединить обе модели в единый класс `Session`
- **C)** Это пример Aggregate Root
- **D)** Это пример Domain Event

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Так в модуле описан **Bounded Context** (Ограниченный контекст) — границы, внутри которых бизнес-термины имеют однозначное значение. В «Обучении» `Skill` — то, чему можно научить, в «Платежах» `Skill` — тарифный план.

**Почему не другие:**
- B — объединение здесь как раз запрещено: модель определяется в рамках своего контекста
- C — Aggregate — это группа сущностей, меняющихся вместе (`User` + `UserSkill`)
- D — Domain Event — это факт «что-то произошло» (`SkillAdded`, `SessionScheduled`)

</details>

---

## Вопрос 3. Разбери фрагмент агрегата из модуля:

```kotlin
class User(
    val id: UserId,
    var name: String,
    private val skills: MutableList<UserSkill> = mutableListOf()
) {
    fun addSkill(skill: Skill, level: SkillLevel) {
        require(!skills.any { it.skillId == skill.id }) {
            "Skill already exists"
        }
        skills.add(UserSkill(skill, level))
    }
}
```

Зачем здесь `require`?

- **A)** Чтобы enforce бизнес-правило «нельзя добавить дубликат» и сохранить инвариант агрегата
- **B)** Это замена `try/catch` для ошибок БД
- **C)** Чтобы проверить, что `skill` не `null`
- **D)** Чтобы опубликовать `DomainEvent`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле это прокомментировано прямо: «Бизнес-логика: нельзя добавить дубликат». `User` — Aggregate Root, а `addSkill` — метод агрегата (не просто setter): он проверяет инвариант и только потом меняет состояние. Поле `skills` приватное, поэтому иначе изменить его нельзя.

**Почему не другие:**
- B — `require` выбрасывает `IllegalArgumentException`, с БД он никак не связан
- C — типы `Skill` и `SkillLevel` не nullable, проверка не нужна
- D — событие публикуется отдельно через `eventBus.publish(...)`

</details>

---

## Вопрос 4. Как в модуле публикуется Domain Event `SkillAdded`?

- **A)** `eventBus.publish(SkillAdded(user.id, skill.id))`
- **B)** `throw SkillAdded(user.id, skill.id)`
- **C)** `Result.Failure(SkillAdded(user.id, skill.id))`
- **D)** `skills.add(SkillAdded(user.id, skill.id))`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Пример из модуля:

```kotlin
sealed class DomainEvent {
    data class SkillAdded(val userId: UserId, val skillId: SkillId) : DomainEvent()
    data class SessionScheduled(val sessionId: SessionId, val date: Instant) : DomainEvent()
}

class SkillRepository {
    fun addSkill(user: User, skill: Skill) {
        // ... сохраняем в БД
        eventBus.publish(SkillAdded(user.id, skill.id))
    }
}
```

Событие — это обычный объект-значение, который публикуется в шину событий **после** сохранения.

**Почему не другие:**
- B — событие не исключение
- C — `Result` из раздела про обработку ошибок, а не про доменные события
- D — в `skills` лежат `UserSkill`, а не события

</details>

---

## Вопрос 5. Граф зависимостей модулей: `apps/* → feature-* → core-* → domain-*`. Что ЗАПРЕЩЕНО правилам из модуля?

- **A)** `feature-auth` зависит от `core-common`
- **B)** `domain-models` зависит от `feature-auth`
- **C)** `apps/skillsync-android` зависит от `feature-skills`
- **D)** `core-data` зависит от `domain-models`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Правило из модуля: **модуль нижнего уровня НЕ может зависеть от модуля верхнего уровня**, обратных зависимостей нет. `domain-models` — самый нижний слой, и он не должен «знать» ни о каких feature-модулях.

**Почему не другие:**
- A, C, D — это как раз разрешённые стрелки вниз по DAG

</details>

---

## Вопрос 6. Где в рекомендуемой структуре SkillSync лежат Ktor Client configuration и entity-классы `User` / `Skill` / `Session`?

- **A)** `core-network` и `domain-models`
- **B)** `core-ui` и `domain-usecases`
- **C)** `core-common` и `core-data`
- **D)** `feature-auth` и `domain-models`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Структура из модуля:

- `shared/core/core-network` — Ktor Client конфигурация
- `shared/domain/domain-models` — Entity-классы (`User`, `Skill`, `Session`)
- `shared/core/core-ui` — общие UI-компоненты; `domain-usecases` — интерфейсы UseCase
- `core-common` — `Result`, `Either`, extensions; `core-data` — общие репозитории и модели

**Почему не другие:**
- B — `core-ui` про кнопки/инпуты, `domain-usecases` про сценарии, а не про entity
- C — оба варианта не совпадают с парой из вопроса
- D — `feature-auth` — фича аутентификации, а не сетевой слой

</details>

---

## Вопрос 7. Что означает запись `kotlin("multiplatform") version "2.0.0" apply false` в root `build.gradle.kts`?

- **A)** Плагин добавляется в classpath, но не применяется к root-проекту; каждый модуль применяет его сам
- **B)** Плагин вообще недоступен модулям
- **C)** Плагин применяется только к Android-модулям
- **D)** Это отключает Version Catalog

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Такая запись — стандарт для root-файла: версия плагина объявляется один раз, но `apply false` не навешивает его на сам root-проект. В `core-common/build.gradle.kts` затем пишется `plugins { kotlin("multiplatform") }` уже без версии.

**Почему не другие:**
- B — ровно наоборот: плагин доступен всем модулям через classpath
- C — `apply false` не накладывает platform-ограничений
- D — Version Catalog живёт в `gradle/libs.versions.toml` и здесь ни при чём

</details>

---

## Вопрос 8. Кастомный `Result<out T, out E>` из `core-common`: что вернёт `flatMap`, если результат — `Failure`?

```kotlin
inline fun <R> flatMap(transform: (T) -> Result<R, E>): Result<R, E> =
    when (this) {
        is Success -> transform(data)
        is Failure -> this as Result<R, E>
    }
```

- **A)** Тот же `Result.Failure` с тем же `error`
- **B)** `Result.Success(null)`
- **C)** Исключение при приведении типа
- **D)** `transform(data)` будет вызван с `null`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Для `Failure` трансформация не выполняется — возвращается сам объект `Failure` (через `this as Result<R, E>`). Это стандартная семантика `flatMap`: цепочка коротко замыкается на первом же провале. Благодаря `out`-variance и `Failure<out E>` приведение безопасно.

**Почему не другие:**
- B — `null` здесь не подставляется нигде
- C — cast корректный: `Failure` не зависит от `T`
- D — `transform` вызывается только в ветке `is Success`

</details>

---

## Вопрос 9. Вопрос-ловушка. Что верно для этого builder'а из модуля?

```kotlin
inline fun <T, E> result(block: () -> T): Result<T, E> =
    try {
        Result.Success(block())
    } catch (e: Exception) {
        @Suppress("UNCHECKED_CAST")
        Result.Failure(e as E)
    }
```

- **A)** `catch` перехватит любой `Throwable`, включая `Error`
- **B)** Если `block` бросит `Error` (не `Exception`), он не будет пойман и уйдёт выше
- **C)** `e as E` всегда безопасен и не может упасть
- **D)** Функция всегда возвращает `Result.Success`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Ловушка на разницу `Exception` vs `Throwable`. В коде ловится именно `Exception`, поэтому `OutOfMemoryError`, `StackOverflowError` и другие `Error` мимо этого `catch` проскочат. Аннотация `@Suppress("UNCHECKED_CAST")` рядом с `e as E` как раз подсказывает, что приведение не проверяется компилятором — оно может упасть, если `E` окажется несовместимым типом.

**Почему не другие:**
- A — для «любого Throwable» нужно было бы писать `catch (e: Throwable)`
- C — `UNCHECKED_CAST` как раз и означает, что проверки нет
- D — при исключении возвращается `Result.Failure`

</details>

---

## Вопрос 10. Где хранится Version Catalog и как в модуле выглядит обращение к `kotlinx-coroutines-core`?

- **A)** `gradle/libs.versions.toml` → `implementation(libs.kotlinx.coroutines.core)`
- **B)** `settings.gradle.kts` → `implementation("@libs/kotlinx-coroutines-core")`
- **C)** корневой `build.gradle.kts` → `implementation(libs.versions.kotlinx)`
- **D)** `gradle.properties` → `implementation("kotlinx.coroutines.core")`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Файл `gradle/libs.versions.toml` содержит секции `[versions]`, `[libraries]`, `[plugins]`. Запись

```toml
kotlinx-coroutines-core = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-core", version.ref = "kotlin" }
```

превращается в accessor `libs.kotlinx.coroutines.core` — дефисы в имени становятся точками, а `version.ref` ссылается на `[versions]`.

**Почему не другие:**
- B — в Gradle TOML-каталоге такого синтаксиса нет, и файл лежит не в `settings.gradle.kts`
- C — `libs.versions...` — это доступ к версии, а не к зависимостям
- D — `gradle.properties` для каталога не используется

</details>

---

## Вопрос 11. Что делает `SkillSyncKmpPlugin` при применении к новому модулю?

```kotlin
class SkillSyncKmpPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        project.plugins.apply("org.jetbrains.kotlin.multiplatform")
        project.extensions.configure<KotlinMultiplatformExtension> {
            androidTarget(); iosX64(); iosArm64(); iosSimulatorArm64(); jvm()
            sourceSets {
                getByName("commonMain") {
                    dependencies { implementation("...kotlinx-coroutines-core:1.7.3") }
                }
            }
        }
    }
}
```

- **A)** Сам применяет KMP-плагин, настраивает таргеты и добавляет общие зависимости в `commonMain`
- **B)** Только создаёт папки для нового feature-модуля
- **C)** Заменяет собой Version Catalog
- **D)** Публикует библиотеку в Maven

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Зачем свой плагин — чтобы не копировать одну и ту же конфигурацию в каждый модуль. После этого в новом модуле достаточно написать:

```kotlin
plugins {
    id("skill-sync-kmp")
}
```

и он сразу получает Android + три iOS-таргета + JVM и coroutines в `commonMain`.

**Почему не другие:**
- B — генерация структуры — отдельная бонусная задача, а не этот плагин
- C — каталог версий живёт в `libs.versions.toml`; плагин лишь использует зависимости
- D — публикация (`publishLibraryVariants`) здесь не настраивается

</details>

---

## Вопрос 12. Какой командой проверить зависимости именно по `commonMainCompileClasspath` у `feature-auth`?

- **A)** `./gradlew :shared:feature:feature-auth:dependencies --configuration commonMainCompileClasspath`
- **B)** `./gradlew clean build`
- **C)** `./gradlew :shared:feature:feature-auth:assemble`
- **D)** `./gradlew --scan`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Эта команда прямо приведена в «Советах по выполнению» модуля. Она показывает граф зависимостей конкретного source set'а — удобно ловить циклы и лишние связи. Отдельно для проверки независимой сборки модуля используется `./gradlew :shared:feature:feature-auth:assemble`, а циклы можно искать и через `./gradlew dependencies`.

**Почему не другие:**
- B — общая сборка не даёт адресного графа
- C — проверяет, что модуль собирается, но не показывает зависимости
- D — build scan не заменяет конкретный `--configuration`

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 10–12 | 🎉 Отлично! DDD, граф зависимостей и Gradle-инструментарий — твоя сильная сторона |
| 7–9 | 📖 Перечитай секции про Version Catalogs и кастомный Gradle-плагин |
| 4–6 | ⚠️ Пройди модуль ещё раз: сфокусируйся на Bounded Context / Aggregate и направлении зависимостей |
| < 4 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🔜 Следующий модуль Senior Track — продвинутый Compose Multiplatform: кастомные Layout'ы, оптимизация и анимации.

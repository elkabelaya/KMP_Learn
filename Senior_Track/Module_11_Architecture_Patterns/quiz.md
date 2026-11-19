# 📝 Самопроверка: Модуль 11 — Архитектурные паттерны

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 12+ из 15 без подглядывания.

> ℹ️ Папка `Module_11_Architecture_Patterns`, но внутри `content.md` этот модуль пронумерован как **Модуль 7** (как и масштабирование/монорепо). Здесь используется номер из самого модуля.

---

## Вопрос 1. Какие проблемы монолита перечисляет модуль и что предлагает взамен?

- **A)** Сложно тестировать, поддерживать и масштабировать, высокая связанность → Clean Architecture с чёткими границами слоёв
- **B)** Монолит плох только тем, что не поддерживает SwiftUI; взамен предлагается Flutter
- **C)** Проблем нет, весь код в `MainActivity` — это и есть рекомендуемая архитектура
- **D)** Главная проблема — медленный интернет; взамен предлагается Ktor Client

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле монолит описан как `MainActivity`, где «всё смешано»: UI, бизнес-логика, доступ к данным, сеть. Следствия: ❌ сложно тестировать, ❌ сложно поддерживать, ❌ сложно масштабировать, ❌ высокая связанность. Решение — Clean Architecture: Presentation / Domain / Data / Platform, где зависимости идут только вниз, к Domain.

**Почему не другие:**
- B — SwiftUI как раз упоминается в слое Presentation как нормальная часть архитектуры
- C — это антипаттерн, с которого модуль начинает повествование
- D — Ktor относится к Data Layer, а не к причинам рефакторинга

</details>

---

## Вопрос 2. Как направлены зависимости между слоями (Dependency Rule)?

- **A)** `Presentation → Domain ← Data`; Platform — точка входа, зависит от всех слоёв; Domain не зависит ни от чего
- **B)** `Domain → Data → Presentation → Platform`: всё зависит от платформы
- **C)** Каждый слой зависит от всех остальных — так проще переиспользовать код
- **D)** Data не зависит ни от чего, а Domain реализует интерфейсы Data

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Схема из модуля: `Presentation → Domain ← Data` и отдельно Platform снизу. Domain («100% KMP») — ядро без зависимостей от других слоёв; Presentation знает Domain через интерфейсы; Data реализует интерфейсы Domain; Platform (Activities, ViewControllers, DI) — точка входа и зависит от всех. Это и есть Dependency Rule: зависимости направлены **внутрь**, к Domain.

**Почему не другие:**
- B — инвертирует правило: Domain не должен зависеть от UI, БД или платформы
- C — это и есть high coupling, от которого модуль уходит
- D — наоборот: контракты (Repository Interfaces) живут в Domain, а реализация — в Data

</details>

---

## Вопрос 3. Где в структуре проекта находятся Entities, Repository Interfaces и Use Cases?

- **A)** В `data/` — рядом с реализациями
- **B)** В `presentation/` — рядом с ViewModel
- **C)** В `domain/` (`models/`, `repository/`, `usecases/`) внутри `commonMain`
- **D)** В `androidMain/` и `iosMain/`, по одной реализации на платформу

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

Структура из модуля:

```
commonMain/kotlin/com/skillsync/
├── domain/         ← Entities (models), Repository Interfaces, Use Cases
├── data/           ← Repository Implementations, local/remote, mappers
└── presentation/   ← ViewModels (MVI), Compose screens
```

Domain полностью кроссплатформенный и не содержит платформенных API.

**Почему не другие:**
- A — в `data/` лежат `SkillsRepositoryImpl`, источники данных и мапперы, а не контракты
- B — в `presentation/` — UI и MVI, бизнес-правила туда не кладут
- D — платформенные модули содержат DI и нативные реализации, но не ядро бизнес-логики

</details>

---

## Вопрос 4. Что даёт модификатор `operator` в объявлении Use Case?

```kotlin
interface GetSkillsUseCase {
    suspend operator fun invoke(): Result<List<Skill>>
}

class GetSkillsUseCase(
    private val repository: SkillsRepository
) : GetSkillsUseCase {
    override suspend fun invoke(): Result<List<Skill>> { /* ... */ }
}
```

- **A)** Позволяет вызывать use case как функцию — `getSkillsUseCase()` вместо `.invoke()`; результат — `Result<List<Skill>>`
- **B)** Автоматически регистрирует класс в Koin
- **C)** Делает `invoke()` доступным только из `commonMain`
- **D)** Превращает `Result` в `List<Skill>` без обработки ошибок

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`suspend operator fun invoke()` — стандартный приём для «вызываемых объектов»: в ViewModel написано `getSkillsUseCase.invoke()`, но благодаря `operator` можно писать и `getSkillsUseCase()`. Use Case возвращает `Result`, то есть либо данные, либо ошибку — это позволяет слою Presentation единообразно обработать Success/Failure.

**Почему не другие:**
- B — DI делается отдельно в Koin-модулях (`single<GetSkillsUseCase> { ... }`), `operator` здесь ни при чём
- C — доступность из source set'ов определяется их расположением, не модификатором
- D — `Result` нужно распаковывать; `operator` этого не делает

</details>

---

## Вопрос 5. ❗Ловушка. Найди ошибку в реализации Use Case из модуля

```kotlin
class GetSkillsUseCase(
    private val repository: SkillsRepository
) : GetSkillsUseCase {
    override suspend fun invoke(): Result<List<Skill>> {
        return try {
            val skills = repository.getSkills()
            val sortedSkills = skills.sortByDescending { it.createdAt }
            Result.success(sortedSkills)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

- **A)** Ошибки нет: `Result<List<Skill>>` — это и есть `List<Skill>`
- **B)** `repository.getSkills()` возвращает `Result<List<Skill>>`, а не список: нужно распаковать результат (`getOrNull()` / `map`) перед `sortByDescending`
- **C)** `sortByDescending` нельзя применять к `Long` — полю `createdAt`
- **D)** Нельзя вызывать `suspend`-функцию внутри `try` — нужно вынести её наружу

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Это вопрос-ловушка: код взят из модуля, но он не компилируется. Domain-интерфейс объявлен как `suspend fun getSkills(): Result<List<Skill>>`, значит `skills` имеет тип `Result<List<Skill>>`, а `sortByDescending` — метод над `Iterable<T>`. Нужно сначала получить список, например `val skills = repository.getSkills().getOrNull()` с обработкой `null`, либо `repository.getSkills().map { list -> list.sortedByDescending { it.createdAt } }`.

**Почему не другие:**
- A — `Result` — обёртка над значением, а не сам список; в коде ниже модуль сам различает `Result.Success` и `Result.Failure`
- C — `createdAt: Long` как раз корректный ключ сортировки; проблема в типе контейнера
- D — `suspend`-вызовы внутри `try` допустимы; в остальных примерах модуля используется та же конструкция

</details>

---

## Вопрос 6. Как `SkillsRepositoryImpl` реализует стратегию cache-then-network?

- **A)** Всегда сначала запрашивает сервер, а кэш использует только при отсутствии сети
- **B)** Непустой кэш → вернуть кэш и обновиться в фоне; пустой кэш → сеть и `saveAll`; при ошибке → старый кэш, если есть, иначе `Result.failure`
- **C)** Непустой кэш → вернуть кэш и никогда не обращаться к сети; при ошибке всегда `Result.failure`
- **D)** Хранит данные только в локальной базе, remote-источник не используется

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Логика `getSkills()` в модуле:

1. `localDataSource.getAllSkills()` → если не пусто: `refreshSkillsInBackground()` и `Result.success(cachedSkills)`;
2. если пусто: `remoteDataSource.getSkills()` → `localDataSource.saveAll(...)` → `Result.success(remoteSkills)`;
3. в `catch`: снова читаем кэш — есть данные → `Result.success(cachedSkills)`, нет → `Result.failure(e)`.

Это даёт быстрый UI и офлайн-фолбэк.

**Почему не другие:**
- A — при непустом кэше сервер не блокирует ответ; обновление уходит в фон
- C — фоновое обновление всё же предусмотрено (`refreshSkillsInBackground`)
- D — remoteDataSource активно используется и в `getSkills`, и в `createSkill`, и в поиске

</details>

---

## Вопрос 7. Что происходит при `createSkill()` в Data Layer и при ошибке сети?

```kotlin
override suspend fun createSkill(skill: Skill): Result<Skill> = try {
    localDataSource.saveSkill(skill)          // 1
    val created = remoteDataSource.createSkill(skill) // 2
    localDataSource.updateSkill(created)      // 3
    Result.success(created)
} catch (e: Exception) {
    localDataSource.deleteSkill(skill.id)     // 4
    Result.failure(e)
}
```

- **A)** Сначала сервер, потом локальная копия; при ошибке локальная запись остаётся
- **B)** Оптимистичная запись локально → синхронизация с сервером → обновление локальной копии ответом сервера; при ошибке локальная запись удаляется (откат)
- **C)** Локальная запись не делается вовсе — это «пессимистичный UI»
- **D)** При ошибке возвращается `Result.success(skill)` с локальным id

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Комментарии в модуле: «Сначала сохраняем локально для оптимистичного UI», затем «синхронизируем с сервером», затем «обновляем локальную копию с данными от сервера». В `catch` — «если ошибка сети, удаляем локальную запись» и `Result.failure(e)`. Это классический optimistic update с rollback.

**Почему не другие:**
- A — порядок шагов 1 и 2 перепутан
- C — как раз локальная запись и делает UI оптимистичным
- D — при ошибке модуль возвращает `Result.failure(e)`, а не успех

</details>

---

## Вопрос 8. Из чего состоит MVI в ViewModel из модуля?

- **A)** `Intent` — `sealed class` действий пользователя; `State` — `data class` (`skills`, `isLoading`, `error`, `searchQuery`, `isSearching` + вычисляемые `isEmpty`/`hasError`); ViewModel на `onIntent` обновляет `StateFlow`
- **B)** `Intent` — это HTTP-запрос; `State` — база данных; ViewModel — Compose-функция
- **C)** `State` хранится в Koin, а `Intent` — в SQLDelight
- **D)** MVI здесь то же самое, что MVVM: Separate Intents не нужны

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В коде модуля:

- `sealed class SkillsIntent` (LoadSkills, SearchSkills, CreateSkill, UpdateSkill, DeleteSkill) — намерения пользователя;
- `data class SkillsUiState` с пятью полями и производными `isEmpty` / `hasError`;
- `SkillsViewModel` держит `MutableStateFlow<SkillsUiState>` и метод `onIntent(intent)` с `when`, который вызывает нужный приватный метод и через `_state.update { it.copy(...) }` меняет состояние.

**Почему не другие:**
- B — это подмена понятий: Ktor/БД живут в Data Layer, Compose — в Presentation, но не являются частями MVI
- C — Koin отвечает за DI, SQLDelight — за хранение данных
- D — модуль явно вводит отдельный поток Intent → State, это и есть отличительная черта MVI

</details>

---

## Вопрос 9. Как в `loadSkills()` обрабатываются состояния загрузки и ошибки?

```kotlin
_state.update { it.copy(isLoading = true, error = null) }
try {
    val result = getSkillsUseCase.invoke()
    _state.update { currentState ->
        when (result) {
            is Result.Success -> currentState.copy(skills = result.data, isLoading = false)
            is Result.Failure -> currentState.copy(
                isLoading = false,
                error = result.exception.message ?: "Unknown error"
            )
        }
    }
} catch (e: Exception) {
    _state.update { it.copy(isLoading = false, error = e.message ?: "Unknown error") }
}
```

- **A)** В начале `isLoading = true` и сбрасывается старый `error`; Success пишет данные и снимает loading; Failure и catch пишут сообщение и тоже снимают loading
- **B)** `isLoading` выставляется только при ошибке
- **C)** При Failure список очищается, а `isLoading` остаётся `true`
- **D)** `catch` нужен только для отмены корутин и не меняет состояние

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Разбор по шагам: стартовая копия состояния включает флаг загрузки и очистку прежней ошибки. Ветка Success кладёт `result.data` в `skills` и выставляет `isLoading = false`. Ветка Failure оставляет старый список, но записывает текст ошибки и тоже завершает загрузку. `catch` — дополнительная страховка от непредвиденных исключений, он повторяет логику Failure с фолбэком `"Unknown error"`.

**Почему не другие:**
- B — флаг выставляется **до** вызова use case, ещё до получения результата
- C — Failure не очищает список: пользователь видит предыдущие данные и ошибку
- D — `catch` меняет и `isLoading`, и `error` — это часть контракта состояния UI

</details>

---

## Вопрос 10. Какие Koin-зависимости подключаются и в каких source set'ах?

- **A)** `koin-core` и `koin-core-coroutines` в common-коде KMP; `koin-android` — только в `androidMain`
- **B)** `koin-android` в `commonMain`, чтобы быть доступным и в iOS
- **C)** Только `koin-android`, остальное Koin генерирует сам
- **D)** Koin в этом модуле не используется — DI реализован вручную

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В `build.gradle.kts` модуля:

```kotlin
implementation("io.insert-koin:koin-core:3.5.0")
implementation("io.insert-koin:koin-core-coroutines:3.5.0")
// Koin для Android (в androidMain):
// implementation("io.insert-koin:koin-android:3.5.0")
```

`koin-android` нужен для Android-специфичных вещей (`androidContext()`, сервисы), поэтому он остаётся в `androidMain`. В feature-модуле та же логика: `libs.koin.core` в `commonMain`, `koin-android` в `androidMain`.

**Почему не другие:**
- B — `koin-android` содержит Android API и недоступен в iOS
- C — KMP-часть работает на `koin-core`; ничего «самогенерирующегося» нет
- D — весь раздел 4 модуля посвящён DI на Koin

</details>

---

## Вопрос 11. Что делают `get()` и `includes()` в Koin-модулях?

```kotlin
val domainModule = module {
    single<GetSkillsUseCase> { GetSkillsUseCase(get<SkillsRepository>()) }
}

val sharedModule = module {
    includes(domainModule, dataModule, presentationModule)
}
```

- **A)** `get()` разрешает зависимость из графа Koin (здесь — `SkillsRepository`), `includes()` объединяет три модуля в один `sharedModule`; `single` создаёт один экземпляр
- **B)** `get()` — это геттер `SkillsViewModel`, `includes()` — аналог Gradle `include()`
- **C)** `single` создаёт новый экземпляр при каждом обращении, `get()` работает только в Android
- **D)** `sharedModule` доступен только в `androidPlatformModule`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В DSL Koin `get<T>()` запрашивает уже зарегистрированную зависимость нужного типа; её регистрирует `dataModule` через `single<SkillsRepository> { SkillsRepositoryImpl(...) }`. `single` означает синглтон — одна инстанция на граф. `includes()` собирает `domainModule`, `dataModule` и `presentationModule` в общий `sharedModule`, который затем входит в `createAndroidModules()` / `createIOSModules()`.

**Почему не другие:**
- B — `get()` не имеет отношения к ViewModel; `includes()` — не Gradle
- C — для нового экземпляра на запрос есть `factory`, а `get()` работает в common-коде на обеих платформах
- D — `sharedModule` входит в обе платформенные комбинации: `listOf(sharedModule, androidPlatformModule)` и `listOf(sharedModule, iosPlatformModule)`

</details>

---

## Вопрос 12. ❗Ловушка. В чём подводный камень expect/actual для платформенных зависимостей?

```kotlin
// commonMain
expect fun getDatabase(): Any
expect fun getHttpClient(): Any

// androidMain
actual fun getDatabase() = TODO("Use Koin injection in Android")
actual fun getHttpClient() = TODO("Use Koin injection in Android")
```

- **A)** Ошибки нет: Koin сам подставит реализацию в `TODO`
- **B)** `actual`-функции — заглушки: при вызове будет `NotImplementedError`. Реальные БД и HTTP-клиенты регистрируются отдельно в `androidPlatformModule` / `iosPlatformModule`
- **C)** `expect` нельзя объявлять в `commonMain` — только в `androidMain`
- **D)** Для `actual` обязателен модификатор `override`, иначе компиляция упадёт

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В модуле `dataModule` вызывает `getDatabase()` и `getHttpClient()`, а platform-модули вместо реальных реализаций содержат `TODO(...)`. `TODO()` в Kotlin бросает `NotImplementedError` — то есть DI-граф соберётся, но первый же вызов упадёт. Настоящие объекты создаются в `androidPlatformModule` (`AndroidDatabaseProvider`, `AndroidHttpClientProvider`) и `iosPlatformModule` (`IOSDatabaseProvider`, `IOSHttpClientProvider`) через Koin; expect/actual здесь нужно либо реализовать по-настоящему, либо не использовать как источник зависимостей.

**Почему не другие:**
- A — Koin не умеет «дописать» тело функции
- C — `expect` как раз объявляется в `commonMain`, а `actual` — в платформенных source set'ах
- D — `actual` не является переопределением, `override` не требуется

</details>

---

## Вопрос 13. Зачем модуль предлагает выносить фичи в отдельные Gradle-модули?

- **A)** Чтобы ускорить сборку, разделить ответственность между командами и не тащить ненужные зависимости; `shared` делится на `core-*`, фичи — в `features/*`, приложения — в `apps/*`
- **B)** Чтобы полностью отказаться от `shared` и писать всё на Swift
- **C)** Чтобы все зависимости подключались в каждый модуль — так надёжнее
- **D)** Чтобы отключить iOS-таргеты и собирать только Android

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В разделе перечислены проблемы монолитного `shared`: медленная сборка при любом изменении, сложно разделить ответственность, все зависимости грузятся, даже если не нужны. Решение: `core-common`, `core-data`, `core-domain`, `core-presentation` плюс `feature-skill-sync`, `feature-ar-card`, `feature-profile`, `feature-settings`, а `apps/*` собирают фичи. Важно не усложнять заранее: «начните с одного feature module, потом масштабируйте».

**Почему не другие:**
- B — KMP-ядро остаётся общим; модулизация не отменяет кроссплатформенность
- C — наоборот, каждая фича подключает только свои `core-*` зависимости
- D — feature-модули объявляют `iosX64()`, `iosArm64()`, `iosSimulatorArm64()`, то есть остаются KMP

</details>

---

## Вопрос 14. Что верно для `build.gradle.kts` feature-модуля?

```kotlin
val commonMain by getting {
    dependencies {
        implementation(project(":shared:core-common"))
        implementation(project(":shared:core-domain"))
        implementation(project(":shared:core-data"))
        implementation(project(":shared:core-presentation"))
        implementation("io.insert-koin:koin-core:3.5.0")
    }
}
val androidMain by getting {
    dependencies { implementation("io.insert-koin:koin-android:3.5.0") }
}
```

- **A)** Фича зависит от `core-*` и koin-core в `commonMain`, `koin-android` изолирован в `androidMain`; модуль — отдельный Gradle-проект с Android и тремя iOS-таргетами
- **B)** `koin-android` следует перенести в `commonMain`, чтобы iOS тоже его использовал
- **C)** `core-*` нужно заменить зависимостью от `apps/skillsync-android`
- **D)** iOS-таргеты конфигурировать не нужно: фича собирается только под Android

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В файле модуля действительно объявлены `androidTarget()` и три iOS-таргета, `commonMain` тянет четыре `core-*` модуля и `koin-core`, а `koin-android` живёт в `androidMain`. Плюс задан `namespace = "com.skillsync.feature.skillssync"`. Это и есть изолированный feature module.

**Почему не другие:**
- B — `koin-android` содержит Android API и сломает iOS-сборку
- C — направление зависимостей обратное: приложение зависит от фич, а не наоборот
- D — без iOS-таргетов фича не будет доступна в iOS-приложении

</details>

---

## Вопрос 15. Что из перечисленного — критерии сдачи домашнего задания модуля?

- **A)** Чёткие слои Domain/Data/Presentation; Domain без зависимостей; MVI ViewModels с StateFlow; Koin работает на обеих платформах; feature module вынесен отдельно; 80%+ coverage для Domain и Presentation
- **B)** 70%+ coverage для `commonMain`, Espresso и XCUITest, CI с GitHub Actions
- **C)** Только Koin на Android и один общий `shared` модуль без feature-изоляции
- **D)** Только Clean Architecture без MVI и DI — остальное бонус

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Список критериев из модуля: ✅ разделение на слои; ✅ Domain не зависит от других слоёв; ✅ MVI ViewModels с StateFlow для всех экранов; ✅ Koin на обеих платформах; ✅ feature module — отдельный Gradle-модуль; ✅ 80%+ coverage для Domain и Presentation. Бонусы: ещё два feature-модуля, offline-first с background sync, поддержка SQLite + Realm.

**Почему не другие:**
- B — Coverage 70% и UI-тесты — это критерии модуля тестирования; здесь порог 80% и упор на архитектуру
- C — игнорирует iOS и модулизацию, оба пункта прямо в критериях
- D — MVI и DI обязательны, а не бонус

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 13–15 | 🎉 Отлично: Clean Architecture, MVI, Koin и модульность уложились в голове |
| 10–12 | 📖 Перечитай секции MVI и DI, сверься со схемой зависимостей слоёв |
| 6–9 | ⚠️ Пройди модуль заново: начни с Domain и Dependency Rule, затем разбери код ViewModel |
| < 6 | 🆘 Вернись к началу: эволюция от монолита и три слоя, потом Koin и feature-модули |

---

> 🔜 Следующий шаг — масштабирование: monorepo, version catalog и convention plugins для больших KMP-проектов.

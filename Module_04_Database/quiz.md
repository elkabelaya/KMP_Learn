# 📝 Самопроверка: Модуль 4

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 11+ из 13 без подглядывания.

---

## Вопрос 1. Почему SQLDelight — рекомендуемый выбор для БД в KMP?

- **A)** Потому что писать SQL-запросы не нужно вообще
- **B)** Типобезопасность (компилятор проверяет SQL), миграции, нативная производительность и KMP-first
- **C)** Потому что он работает только на Android и хорошо интегрирован с `Activity`
- **D)** Потому что он заменяет ViewModel и StateFlow

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В таблице модуля SQLDelight помечен ✅ «Рекомендуется», а причины перечислены отдельно: типобезопасность, миграции, нативная производительность (SQLite на Android, SQLite/CoreData на iOS), KMP-first.

Альтернативы: Room Multiplatform — знакомый API, но меньше фич; Realm — быстрая, но проприетарная лицензия; связка CoreData (iOS) + Room (Android) — дублирование кода, ❌ не рекомендуется.

**Почему другие не подходят:**
- A — как раз наоборот: SQL-запросы писать нужно (в `.sq`-файлах)
- C — SQLDelight кроссплатформенный, драйверы есть и для Android, и для iOS
- D — БД не заменяет ViewModel: ViewModel по-прежнему управляет состоянием экрана

</details>

---

## Вопрос 2. Где лежит файл схемы и куда генерируется код SQLDelight?

- **A)** `src/commonMain/sqldelight/com/ecotrack/db/Habit.sq`; сгенерированный код — в `shared/build/generated/sqldelight/`
- **B)** `src/androidMain/sqldelight/...`; код генерируется в iOS-фреймворке
- **C)** В `gradle/libs.versions.toml`, код генерируется в `build.gradle.kts`
- **D)** В `AndroidManifest.xml`, код — в `MainActivity`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

По модулю: `sqldelight` — папка, в которой плагин ищет описание БД, а `com/ecotrack/db` — это `packageName`, заданный в блоке `sqldelight { databases { create("EcoTrackDatabase") { ... } } }` в Gradle.

Порядок работы: пишешь SQL в `.sq` → Gradle компилирует SQL в Kotlin-код → сгенерированный код лежит в `shared/build/generated/sqldelight/`.

**Почему другие не подходят:**
- B — схема общая и лежит в `commonMain`, а не в Android
- C — каталог версий не имеет отношения к файлам схемы
- D — манифест и Activity здесь ни при чём

</details>

---

## Вопрос 3. Какой набор зависимостей и настроек для SQLDelight приведён в модуле?

- **A)** Плагин `app.cash.sqldelight:2.3.2`; `runtime` в `commonMain`; `sqlite-driver` в `androidMain`; `native-driver` в `iosMain`; для iOS-фреймворка `linkerOpts("-lsqlite3")`
- **B)** `runtime` только в `androidMain`, `native-driver` в `commonMain`
- **C)** Достаточно одного `sqlite-driver` в `commonMain` для всех платформ
- **D)** Плагин подключать не нужно, достаточно `kotlinx-serialization`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Конфигурация из модуля:

```kotlin
plugins {
    id("app.cash.sqldelight:2.3.2")
}

// iOS-таргеты: framework { baseName = "Shared"; isStatic = true; linkerOpts("-lsqlite3") }

commonMain.dependencies  { implementation("app.cash.sqldelight:runtime:2.3.2") }
androidMain.dependencies { implementation("app.cash.sqldelight:sqlite-driver:2.3.2") }
iosMain.dependencies     { implementation("app.cash.sqldelight:native-driver:2.3.2") }
```

Разные драйверы нужны потому, что под Android и под iOS используются разные реализации SQLite.

**Почему другие не подходят:**
- B — `runtime` нужен в commonMain, а `native-driver` — в iOS
- C — универсального драйвера в commonMain нет: нужны platform-specific `actual`
- D — без плагина `.sq`-файлы не будут компилироваться в Kotlin

</details>

---

## Вопрос 4. Какой драйвер создать в `iosMain` и что учесть в конфигурации фреймворка?

- **A)** `AndroidSqliteDriver(schema = EcoTrackDatabase.Schema, context = ..., name = Database.FILENAME)` и ничего больше
- **B)** `NativeSqliteDriver(schema = EcoTrackDatabase.Schema.synchronous(), name = Database.FILENAME)`; для iOS-фреймворка добавить `linkerOpts("-lsqlite3")`
- **C)** `SqliteDriver()` без параметров — драйвер определит платформу сам
- **D)** Драйвер создаётся только в `commonMain` один раз на все платформы

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Код из `iosMain`:

```kotlin
fun createDriver(): SqlDriver = NativeSqliteDriver(
    schema = EcoTrackDatabase.Schema.synchronous(),
    name = Database.FILENAME
)
```

А для iOS-таргетов в Gradle указано:

```kotlin
iosTarget.binaries.framework {
    baseName = "Shared"
    isStatic = true
    linkerOpts("-lsqlite3")
}
```

`Context` нужен только Android-драйверу.

**Почему другие не подходят:**
- A — это Android-вариант, на iOS его нет
- C — у драйвера обязательны схема и имя файла БД
- D — драйверы платформенные; в commonMain живут только SQL и общая логика

</details>

---

## Вопрос 5. Что произойдёт после изменения строки в базе, если репозиторий написан так?

```kotlin
override fun getHabits(): Flow<List<HabitEntity>> {
    return database.habitQueries
        .getAllHabits()
        .asFlow()
        .mapToList(Dispatchers.Default)
        .map { list -> list.map { it.toDomainEntity() } }
}
```

- **A)** Ничего: нужно вручную вызвать `loadHabits()`
- **B)** `Flow` испустит новый список, ViewModel обновит `uiState`, и список на экране обновится сам
- **C)** Приложение упадёт с ошибкой транзакции
- **D)** Изменение не отразится до перезапуска приложения

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`asFlow()` превращает запрос SQLDelight в `Flow`, `mapToList(Dispatchers.Default)` собирает строки в список вне главного потока, а `.map { ... toDomainEntity() }` переводит строки БД в бизнес-сущности. Так как это `Flow`, изменение данных автоматически приведёт к новому значению.

В ViewModel (модуль 4) именно это и используется:

```kotlin
repository.getHabits().collect { habits ->
    _uiState.value = _uiState.value.copy(isLoading = false, habits = habits)
}
```

Один из критериев сдачи домашнего задания: «Список обновляется в реальном времени (Flow)».

**Почему другие не подходят:**
- A — ручной `loadHabits()` нужен после записи (`addHabit`/`deleteHabit`), но не для реактивного чтения
- C — чтение через `Flow` не требует транзакции
- D — это противоречит смыслу `Flow` и критерию сдачи

</details>

---

## Вопрос 6. Почему при вставке `category` превращают в `ordinal.toLong()`, а `isCompleted` — в `1L/0L`?

```kotlin
database.habitQueries.insertHabit(
    id = habit.id,
    title = habit.title,
    category = habit.category.ordinal.toLong(),
    isCompleted = if (habit.isCompleted) 1L else 0L,
    createdAt = habit.createdAt,
    updatedAt = System.currentTimeMillis()
)
```

- **A)** Потому что в схеме `category INTEGER` (0=TRANSPORT, 1=WASTE, 2=WATER, 3=PLASTIC) и `is_completed INTEGER NOT NULL DEFAULT 0`
- **B)** Потому что SQLDelight не умеет работать со строками
- **C)** Потому что `category` на самом деле `String`, а `isCompleted` — `Int`
- **D)** Это ошибка: нужно передавать `category.name` и сам `Boolean`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В `.sq`-схеме колонки объявлены как `category INTEGER NOT NULL` с комментарием «0=TRANSPORT, 1=WASTE, 2=WATER, 3=PLASTIC» и `is_completed INTEGER NOT NULL DEFAULT 0`. SQLite хранит эти значения целыми числами, поэтому enum переводят в порядковый номер, а Boolean — в 1/0.

Обратный маппинг в `toDomainEntity()`: `HabitCategory.entries[this.category.toInt()]` и `isCompleted = this.is_completed == 1L`.

**Почему другие не подходят:**
- B — SQLDelight работает и со строками (`title TEXT`), дело в схеме
- C — в Domain `category` это enum, `isCompleted` — Boolean
- D — передача `category.name`/Boolean не соответствует типу `INTEGER` в сгенерированном API

</details>

---

## Вопрос 7. Вопрос-ловушка. Найди ошибку в iOS-драйвере:

```kotlin
fun createDriver(): SqlDriver = NativeSqliteDriver(
    schema = EcoTrackDatabase.Schema,
    name = Database.FILENAME
)
```

- **A)** Ничего, код корректный
- **B)** На iOS нужно передать `EcoTrackDatabase.Schema.synchronous()`
- **C)** Нужно обязательно передать `Context`, как на Android
- **D)** Нужно заменить `NativeSqliteDriver` на `AndroidSqliteDriver`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Ловушка в том, что фрагмент почти совпадает с правильным iOS-кодом из модуля, но упускает `.synchronous()`:

```kotlin
NativeSqliteDriver(
    schema = EcoTrackDatabase.Schema.synchronous(),
    name = Database.FILENAME
)
```

Дополнительно для iOS-фреймворка в Gradle нужен `linkerOpts("-lsqlite3")`.

**Почему другие не подходят:**
- A — отличие от эталонного кода есть
- C — `Context` — это Android-specific API, его нет в Kotlin/Native
- D — `AndroidSqliteDriver` используется в `androidMain`, а не в `iosMain`

</details>

---

## Вопрос 8. Зачем операции записи обёрнуты в `withContext(Dispatchers.Default)`?

- **A)** Чтобы получить доступ к `Context` Android
- **B)** Чтобы выполнить синхронную работу с SQLite вне главного потока
- **C)** Чтобы автоматически создать транзакцию
- **D)** Чтобы переключиться на главный поток для обновления UI

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В репозитории из модуля `addHabit`, `updateHabit` и `deleteHabit` объявлены как `= withContext(Dispatchers.Default) { ... }`, а чтение использует `.mapToList(Dispatchers.Default)`. Вызовы БД синхронные, поэтому их выносят с main thread. Транзакция — отдельный вызов `database.transaction { ... }` (есть в `addHabit` и `updateHabit`), а не следствие `withContext`.

**Почему другие не подходят:**
- A — `Context` нужен только при создании `AndroidSqliteDriver`
- C — транзакция создаётся явно через `database.transaction { }`
- D — `Dispatchers.Default` — это фоновый пул, а не главный поток

</details>

---

## Вопрос 9. Как `toDomainEntity()` переводит строку БД в `HabitEntity`?

```kotlin
private fun Habit.toDomainEntity(): HabitEntity = HabitEntity(
    id = this.id,
    title = this.title,
    category = HabitCategory.entries[this.category.toInt()],
    isCompleted = this.is_completed == 1L,
    createdAt = this.created_at
)
```

- **A)** `is_completed == 1L` превращает целое число в Boolean, а `entries[...]` — число в enum
- **B)** `isCompleted` всегда `true`, а категория берётся из `title`
- **C)** Преобразование не нужно, можно вернуть `Habit` как есть
- **D)** Категория вычисляется через `HabitCategory.values()[category]`, а `isCompleted` через `toBoolean()`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Как раз обратная сторона маппинга из вопроса 6: `category` (INTEGER) → `HabitCategory.entries[category.toInt()]`, `is_completed` (INTEGER 0/1) → Boolean через сравнение с `1L`. Это extension-функция в commonMain, отделяющая модель БД от Domain-сущности.

**Почему другие не подходят:**
- B — это потеря данных
- C — `Habit` (сгенерированный из БД тип) и `HabitEntity` (Domain) — разные типы, репозиторий обязан вернуть `List<HabitEntity>`
- D — в модуле используется именно `entries[...]` и сравнение `== 1L`

</details>

---

## Вопрос 10. Откуда `EcoTrackDatabase` и `HabitRepository` получают свои зависимости?

```kotlin
fun databaseModule(sqlDriver: SqlDriver) = module {
    single<SqlDriver> { sqlDriver }
}

val commonModule = module {
    single { EcoTrackDatabase(get()) }
    single<HabitRepository> { HabitRepositoryImpl(get()) }
    factory { AddHabitUseCase(get()) }
    factory { GetHabitsUseCase(get()) }
    viewModel { HabitViewModel(get(), get()) }
}

fun initKoin(sqlDriver: SqlDriver, config: KoinAppDeclaration? = null) {
    startKoin {
        config?.invoke(this)
        modules(databaseModule(sqlDriver), platformModule, commonModule)
    }
}
```

- **A)** `single<SqlDriver>` из `databaseModule` передаётся в `EcoTrackDatabase(get())`, затем база — в `HabitRepositoryImpl(get())`, а UseCases и ViewModel собираются из них же
- **B)** Они создаются напрямую в `HomeScreen`
- **C)** Их передаёт Swift в `initKoinIOS()`
- **D)** Они генерируются из `.sq`-файла

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Это продолжение Koin из модуля 3: платформа создаёт `SqlDriver` и отдаёт его в `initKoin`, `databaseModule` регистрирует его как `single<SqlDriver>`, `commonModule` строит из него `EcoTrackDatabase`, затем `HabitRepositoryImpl`, UseCases и ViewModel. Все модули подключаются в `modules(databaseModule(sqlDriver), platformModule, commonModule)`.

**Почему другие не подходят:**
- B — Composable ничего не создаёт, он только наблюдает за ViewModel
- C — Swift передаёт только драйвер (`createDriver()`), а не репозиторий
- D — `.sq` генерирует сгенерированные классы БД, но не граф зависимостей Koin

</details>

---

## Вопрос 11. Как инициализируется Koin с драйвером на Android и на iOS?

- **A)** Android: `initKoin(createDriver(this)) { androidContext(this@AndroidApp) }`; iOS: `initKoinIOS() { initKoin(createDriver()) }`
- **B)** Обе платформы вызывают `createDriver(context)` с одним и тем же параметром
- **C)** Только Android создаёт драйвер, iOS получает его из Koin автоматически
- **D)** Драйвер создаётся в `commonMain` при старте приложения

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В `AndroidApp.kt`:

```kotlin
initKoin(createDriver(this)) {
    androidContext(this@AndroidApp)
}
```

Здесь `createDriver(context)` — это `AndroidSqliteDriver`, а `androidContext(...)` — платформенная конфигурация из модуля 3.

В iOS:

```kotlin
fun initKoinIOS() {
    initKoin(createDriver())
}
```

`createDriver()` без аргументов — `NativeSqliteDriver`.

**Почему другие не подходят:**
- B — у iOS нет Android `Context`
- C — iOS тоже обязан создать свой драйвер
- D — драйверы объявлены в платформенных source set'ах, не в commonMain

</details>

---

## Вопрос 12. Как добавить в базу новое поле `description` через миграцию?

- **A)** Добавить пронумерованный SQL-файл `2_add_description.sql` с `ALTER TABLE habit ADD COLUMN description TEXT;` — SQLDelight применит миграции по порядку
- **B)** Удалить старую БД и создать заново без миграций
- **C)** Написать миграцию на Kotlin в ViewModel
- **D)** Просто изменить `.sq`-файл с запросами, ничего больше не делая

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле показаны файлы `src/commonMain/sq/com/ecotrack/db/1_create_habit_table.sql` (создание таблицы и индекса) и `2_add_description.sql`:

```sql
-- Версия 2: Добавляем поле description (если нужно в будущем)
ALTER TABLE habit ADD COLUMN description TEXT;
```

«SQLDelight автоматически применяет миграции по порядку» — то есть по номерам файлов.

**Почему другие не подходят:**
- B — при таком подходе пользователь потеряет данные
- C — миграции описываются SQL-скриптами, а не кодом ViewModel
- D — одного изменения запросов недостаточно: нужна смена схемы БД

</details>

---

## Вопрос 13. Что проверяет домашнее задание модуля и о чём предупреждают советы в конце?

- **A)** БД создаётся при первом запуске, привычки сохраняются после перезапуска, список обновляется в реальном времени, ошибки обрабатываются; параметры передают через `?`, а для iOS-симулятора после смены схемы может потребоваться перезапуск
- **B)** Достаточно просто запустить приложение на Android-эмуляторе
- **C)** Значения можно подставлять прямо в SQL-строку — это надёжнее `?`
- **D)** Перед каждым запуском нужно удалять файл базы данных

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Критерии сдачи: «БД создается автоматически при первом запуске», «Привычки сохраняются после перезапуска приложения», «Список обновляется в реальном времени (Flow)», «Ошибки обрабатываются корректно».

Советы: «SQLDelight очень строгий к типам данных. Используйте `?` для параметров, а не вставляйте значения напрямую» и «Для iOS-симулятора может потребоваться перезапуск симулятора после изменения схемы БД».

**Почему другие не подходят:**
- B — этого мало: нужна проверка персистентности и Flow
- C — прямо противоречит совету модуля
- D — база должна сохраняться, а не удаляться

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 11–13 | 🎉 Отлично! SQLDelight, драйверы и Flow поняты — можно идти к Модулю 5 (сеть) |
| 8–10 | 📖 Хорошо. Перечитай секции про драйверы (`Schema.synchronous()`, `linkerOpts`) и маппинг в `toDomainEntity()` |
| 5–7 | ⚠️ Пройди модуль ещё раз: схема `.sq`, CRUD-операции и подключение БД к Koin |
| < 5 | 🆘 Начни с установки SQLDelight и шага 4 (репозиторий), затем вернись к квизу |

---

> 🔜 В Модуле 5 подключим Ktor Client и синхронизацию данных — там снова понадобятся Flow, coroutines и слои из Модуля 3.

# 📝 Самопроверка: Модуль 3

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 12+ из 14 без подглядывания.

---

## Вопрос 1. В чём главная проблема «спагетти-кода» из начала модуля?

```kotlin
@Composable
fun HomeScreen() {
    var habits by remember { mutableStateOf(listOf()) }
    Button(onClick = {
        val newHabit = saveToDatabase(title)
        syncWithServer(newHabit)
        habits += newHabit
    }) { Text("Добавить") }
}
```

- **A)** Composable-функции не умеют вызывать БД
- **B)** Логику нельзя протестировать отдельно от UI, она дублируется и код сложно поддерживать
- **C)** В Compose нельзя использовать `remember`
- **D)** Такой код не скомпилируется в KMP

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В модуле перечислены проблемы спагетти-кода: «Нельзя протестировать логику отдельно от UI», «Код дублируется при изменении требований», «Сложно поддерживать и масштабировать». В примере внутри `onClick` смешаны валидация, запрос к БД и сетевой вызов.

**Почему другие не подходят:**
- A — технического ограничения тут нет, проблема именно в смешивании ответственностей
- C — `remember` как раз используется, он ни при чём
- D — код скомпилируется; беда в архитектуре, а не в синтаксисе

</details>

---

## Вопрос 2. Что входит в Domain-слой Clean Architecture?

- **A)** Compose-экраны и ViewModels
- **B)** UseCases и интерфейсы репозиториев — чистый Kotlin, 100% KMP
- **C)** Реализации БД и сетевые клиенты
- **D)** `MainActivity` и `MainViewController`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Схема из модуля:

```
UI Layer      ← Compose Screens, ViewModels
Domain Layer  ← UseCases, Интерфейсы (100% KMP)
Data Layer    ← Repository, БД, API (Expect/Actual)
```

Domain содержит `HabitEntity`, `HabitCategory`, интерфейс `HabitRepository` и UseCases (`AddHabitUseCase`). Он не зависит от платформ.

**Почему другие не подходят:**
- A — это UI-слой
- C — это Data-слой
- D — это платформенные точки входа, к Domain отношения не имеют

</details>

---

## Вопрос 3. Какое направление зависимостей верно?

- **A)** UI → Data → Domain (UI напрямую работает с БД)
- **B)** UI знает только о Domain (через ViewModel); Domain не зависит от платформ; Data реализует интерфейсы из Domain
- **C)** Domain зависит от UI и от Data
- **D)** Все слои зависят друг от друга без исключений

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Три принципа из модуля:
1. UI знает только о Domain (через ViewModel)
2. Domain не зависит от платформ (только интерфейсы)
3. Data реализует интерфейсы из Domain для каждой платформы

Именно поэтому в домашнем задании просят удалить все зависимости из `HomeScreen`, кроме ViewModel: если код компилируется — архитектура правильная.

**Почему другие не подходят:**
- A — это и есть спагетти/нарушение слоёв: UI не должен знать о БД
- C — Domain должен быть независимым «ядром»
- D — вот как раз этого и нужно избегать

</details>

---

## Вопрос 4. MVVM vs MVI: что рекомендует модуль для KMP и когда используют альтернативу?

- **A)** Рекомендуется MVI, а MVVM — устаревший паттерн
- **B)** Рекомендуется MVVM (View → ViewModel со StateFlow → Model/Repository); MVI — альтернатива для сложных приложений с множеством состояний, где View шлёт Intent, а ViewModel возвращает иммутабельный State
- **C)** Для KMP подходят только паттерны без ViewModel
- **D)** MVI и MVVM — одно и то же

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Модуль прямо называет MVVM «Рекомендуемый для KMP»: ViewModel хранит состояние в `StateFlow`, логика в ViewModel, репозиторий — контракт данных. Преимущества: простота, отличная поддержка в Compose (`collectAsState`), лёгкое тестирование.

Про MVI сказано: View отправляет Intent, ViewModel обрабатывает его и возвращает иммутабельный State; использовать для сложных приложений с множеством состояний. «Для EcoTrack достаточно MVVM».

**Почему другие не подходят:**
- A — наоборот: MVVM рекомендуется
- C — ViewModel как раз центральная часть рекомендованного подхода
- D — это разные паттерны с разной моделью взаимодействия

</details>

---

## Вопрос 5. Чем отличается `HabitEntity` от «модели БД» и что внутри `HabitCategory`?

- **A)** `HabitEntity` — чистая бизнес-модель (`id` по умолчанию `UUID.randomUUID()`, `createdAt` = `System.currentTimeMillis()`), а `HabitCategory` — enum из 4 значений: `TRANSPORT`, `WASTE`, `WATER`, `PLASTIC`
- **B)** `HabitEntity` — это таблица SQLDelight, а `HabitCategory` — класс ViewModel
- **C)** `HabitEntity` хранит только `title`, а категории нет
- **D)** `HabitCategory` — это список строк из UI

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Код из `domain/model/HabitEntity.kt`:

```kotlin
data class HabitEntity(
    val id: String = UUID.randomUUID().toString(),
    val title: String,
    val category: HabitCategory,
    val isCompleted: Boolean = false,
    val createdAt: Long = System.currentTimeMillis()
)

enum class HabitCategory { TRANSPORT, WASTE, WATER, PLASTIC }
```

Комментарий в модуле: «Entity — это чистая бизнес-модель, не зависит от БД».

**Почему другие не подходят:**
- B — таблица появится только в модуле 4, и это уже Data-слой
- C — у сущности есть id, category, isCompleted, createdAt
- D — категория это enum, а не строки интерфейса

</details>

---

## Вопрос 6. Вопрос-ловушка. Что вернёт `AddHabitUseCase` при вызове `invoke("   ", HabitCategory.WASTE)`?

```kotlin
suspend operator fun invoke(
    title: String,
    category: HabitCategory
): Result<HabitEntity> {
    if (title.isBlank()) {
        return Result.failure(Exception("Название не может быть пустым"))
    }
    if (title.length > 100) {
        return Result.failure(Exception("Название слишком длинное"))
    }
    val habit = HabitEntity(title = title.trim(), category = category)
    return repository.addHabit(habit).map { habit }
}
```

- **A)** `Result.success` с привычкой, у которой `title == ""`
- **B)** `Result.success`, потому что `trim()` делает строку валидной
- **C)** `Result.failure(Exception("Название не может быть пустым"))`
- **D)** `NullPointerException`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

Ловушка в том, что `"   "` — не пустая строка (`isNotEmpty()` дал бы `true`), но `isBlank()` возвращает `true`, потому что в ней только пробелы. Валидация выполняется **до** `trim()`, поэтому сразу возвращается failure с сообщением «Название не может быть пустым».

Для строки длиннее 100 символов было бы сообщение «Название слишком длинное». Если валидация пройдена, создаётся `HabitEntity(title = title.trim(), ...)` и результат `repository.addHabit(habit)` мапится в саму привычку через `.map { habit }`.

**Почему другие не подходят:**
- A и B — `trim()` вызывается только после проверок, до репозитория дело не доходит
- D — здесь нет nullable-значений, исключения не будет

</details>

---

## Вопрос 7. Какая сигнатура соответствует контракту `HabitRepository` из Domain?

- **A)** `fun getHabits(): List<HabitEntity>` и `fun addHabit(habit: HabitEntity): Unit`
- **B)** `fun getHabits(): Flow<List<HabitEntity>>` и `suspend fun addHabit(habit: HabitEntity): Result<Unit>`
- **C)** `suspend fun getHabits(): HabitEntity` и `fun addHabit(habit: HabitEntity): Boolean`
- **D)** `fun getHabits(): LiveData<List<HabitEntity>>` и `suspend fun addHabit(habit: HabitEntity): Result<HabitEntity>`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Интерфейс из модуля:

```kotlin
interface HabitRepository {
    fun getHabits(): Flow<List<HabitEntity>>
    suspend fun addHabit(habit: HabitEntity): Result<Unit>
    suspend fun updateHabit(habit: HabitEntity): Result<Unit>
    suspend fun deleteHabit(id: String): Result<Unit>
    suspend fun getHabitsByCategory(category: HabitCategory): Flow<List<HabitEntity>>
}
```

Чтение — `Flow` (реактивные обновления), запись — `suspend` + `Result<Unit>`.

**Почему другие не подходят:**
- A — нет `Flow` и `suspend`/`Result`, а именно это ждёт ViewModel
- C — тип возврата и сигнатура не совпадают с контрактом
- D — `LiveData` Android-specific и не используется в KMP-контракте; `Result<HabitEntity>` не соответствует `Result<Unit>`

</details>

---

## Вопрос 8. Что делает `get()` в объявлении Koin-модуля?

```kotlin
val commonModule = module {
    factory { AddHabitUseCase(get()) }
    viewModel { HabitViewModel(get()) }
}
```

- **A)** Создаёт новый `HabitRepository` вручную внутри UseCase
- **B)** Запрашивает у графа Koin уже объявленную зависимость (здесь — `AddHabitUseCase`) и внедряет её в конструктор
- **C)** Читает значение из `SharedPreferences`
- **D)** Делает сетевой запрос

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В модуле показано: `class HabitViewModel(private val repository: HabitRepository) { ... }` с комментарием «Внедряется автоматически», а в Koin — `viewModel { HabitViewModel(get()) }`. То есть `get()` разрешает зависимость из контейнера Koin. Это и есть DI: зависимости создаются снаружи, а не внутри класса.

**Почему другие не подходят:**
- A — как раз наоборот: UseCase получает репозиторий, а не создаёт его
- C, D — Koin не работает ни с `SharedPreferences`, ни с сетью «из коробки» через `get()`

</details>

---

## Вопрос 9. Как правильно объявить платформенный DI-модуль через expect/actual?

- **A)** В `commonMain`: `expect val platformModule: Module`; в `androidMain` и `iosMain`: `actual val platformModule: Module = module { ... }`
- **B)** Только в `androidMain`, iOS подключит его автоматически
- **C)** Обычный интерфейс в `commonMain` и две реализации без expect/actual
- **D)** Модуль объявляется в Swift (`iOSApp.swift`)

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Код из модуля:

```kotlin
// commonMain
expect val platformModule: Module

// androidMain
actual val platformModule: Module = module { /* Android-специфичные зависимости */ }

// iosMain
actual val platformModule: Module = module { /* iOS-специфичные зависимости (CoreData, URLSession) */ }
```

Затем `appModules = listOf(commonModule, platformModule)` и `initKoin { modules(appModules) }`.

**Почему другие не подходят:**
- B — без `actual` для iOS компиляция iOS-таргета упадёт
- C — для DI в модуле выбран именно механизм expect/actual
- D — Kotlin-модуль объявляется на Kotlin, Swift лишь его инициализирует

</details>

---

## Вопрос 10. Почему для KMP выбран Koin?

- **A)** Потому что он требует много кодогенерации
- **B)** Он легковесный (нет кодогенерации), имеет полную поддержку KMP и простой синтаксис
- **C)** Он работает только на Android
- **D)** Он встроен в Kotlin и не требует зависимостей

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Модуль называет три причины: «Легковесный (нет кодогенерации)», «Полная поддержка KMP», «Простой синтаксис». Зависимости подключаются как `koin-core:4.0.0`, `koin-compose:4.0.0`, `koin-compose-viewmodel:4.0.0` в `commonMain` и `koin-core`/`koin-android` в androidApp.

**Почему другие не подходят:**
- A — наоборот, отсутствие кодогенерации — плюс
- C — Koin как раз кроссплатформенный
- D — зависимости всё-таки нужно добавить в Gradle

</details>

---

## Вопрос 11. Зачем в ViewModel делают `_uiState` приватным и публикуют `asStateFlow()`?

```kotlin
private val _uiState = MutableStateFlow(HabitUiState())
val uiState: StateFlow<HabitUiState> = _uiState.asStateFlow()
```

- **A)** Чтобы UI мог напрямую менять состояние
- **B)** Чтобы наружу был доступен только read-only `StateFlow`, а изменение состояния оставалось внутри ViewModel
- **C)** Чтобы состояние автоматически сохранялось в БД
- **D)** Чтобы не нужно было вызывать `collectAsState()` в Compose

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Мутабельный поток спрятан (`private`), а наружу отдаётся неизменяемый интерфейс `StateFlow`. ViewModel меняет состояние через `_uiState.value = _uiState.value.copy(...)`, а экран просто наблюдает: `val uiState by viewModel.uiState.collectAsState()` (код `HomeScreen` из модуля).

**Почему другие не подходят:**
- A — именно этого мы и избегаем: UI не должен писать в состояние напрямую
- C — сохранение в БД — задача Data-слоя (модуль 4)
- D — `collectAsState()` всё равно нужен, `asStateFlow` его не заменяет

</details>

---

## Вопрос 12. Что нужно сделать дополнительно, чтобы Android инициализировал Koin?

```kotlin
class AndroidApp : Application() {
    override fun onCreate() {
        super.onCreate()
        initKoin {
            androidContext(this@AndroidApp)
        }
    }
}
```

- **A)** Указать `android:name=".AndroidApp"` в `AndroidManifest.xml`
- **B)** Добавить `@Serializable` к `AndroidApp`
- **C)** Вызвать `rememberNavController()` в `onCreate`
- **D)** Ничего, `Application` подхватывается автоматически

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле рядом с кодом `AndroidApp` показан манифест:

```xml
<application
    android:name=".AndroidApp"
    ... />
```

Сам `initKoin(config)` устроен так: `config?.invoke(this)` — это как раз возможность платформы передать свой `Context` через `androidContext(...)`. Без регистрации класса в манифесте `onCreate()` не вызовется.

**Почему другие не подходят:**
- B — `@Serializable` относится к навигации
- C — `NavController` живёт в Compose, а не в `Application`
- D — Android требует явной регистрации `Application` в манифесте

</details>

---

## Вопрос 13. Почему в Swift-коде инициализация Koin выглядит как `AppModuleKt.doInitKoin()`?

```swift
@main
struct iOSApp: App {
    init() {
        AppModuleKt.doInitKoin()
    }
    var body: some Scene { WindowGroup { ContentView() } }
}
```

- **A)** Это имя Swift-протокола для DI
- **B)** Kotlin-файл с top-level функцией (`AppModule.kt`) компилируется в класс `AppModuleKt`, и Swift вызывает оттуда `doInitKoin()`
- **C)** Так называется фреймворк `shared`
- **D)** Это имя ViewModel, созданной в Koin

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Комментарий прямо в модуле: «Название класса формируется из имени Kotlin файла (AppModulet.kt -> AppModuleKt)». Top-level функция `initKoin` в файле `AppModule.kt` видна из Swift как `AppModuleKt.doInitKoin()`. Вызов помещён в `init()` структуры `iOSApp`, которая срабатывает при старте приложения.

**Почему другие не подходят:**
- A — это не Swift-протокол, а сгенерированный Kotlin-класс
- C — фреймворк импортируется как `import Shared`, но вызов идёт к конкретному классу
- D — ViewModel здесь ни при чём

</details>

---

## Вопрос 14. Проверка из домашнего задания: ты удалил из `HomeScreen` все зависимости, кроме ViewModel, и код всё ещё компилируется. Что это значит?

- **A)** Compose-экран сломан и его нужно вернуть
- **B)** Архитектура правильная: UI не знает ни о БД, ни о сети, ни о конкретной реализации репозитория
- **C)** Koin настроен неправильно: нужно внедрять зависимости прямо в Composable
- **D)** Пора удалить ViewModel тоже

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Задание 5 формулируется так: «Попробуйте удалить все зависимости из `HomeScreen` (кроме ViewModel). Убедитесь, что код компилируется — это значит архитектура правильная». Это практическая проверка принципа «UI знает только о Domain через ViewModel».

Критерий сдачи: код разделён на 3 слоя, Koin внедряет зависимости, UseCases содержат валидацию, ViewModel управляет состоянием через StateFlow.

⚠️ Модуль также предупреждает: самая частая ошибка — не все интерфейсы Domain реализованы в Data-слое, из-за чего код не компилируется.

**Почему другие не подходят:**
- A — компиляция как раз подтверждает корректность
- C — как раз наоборот, Koin внедряет ViewModel, а не зависимости в сам Composable
- D — ViewModel — единственное легальное окно UI в Domain, удалять его нельзя

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 12–14 | 🎉 Отлично! Слои, Koin и StateFlow поняты — переходи к Модулю 4 (SQLDelight) |
| 9–11 | 📖 Неплохо. Перечитай секции про Koin (`get()`, `expect/actual`) и про сигнатуры репозитория |
| 5–8 | ⚠️ Пройди модуль ещё раз, сфокусируйся на разделении UI / Domain / Data и на валидации в UseCase |
| < 5 | 🆘 Начни с принципов Clean Architecture и домашнего задания 5, затем вернись к квизу |

---

> 🔜 В Модуле 4 заглушка репозитория заменится настоящей базой SQLDelight — вопросы про `.sq`-файлы, драйверы и миграции.

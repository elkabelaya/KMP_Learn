# 📝 Самопроверка: Модуль 8

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 12+ из 15 без подглядывания.

---

## Вопрос 1. Какой инструмент в таблице модуля соответствует E2E-тестированию на реальных устройствах?

- **A)** JUnit 5 и Kotest
- **B)** Maestro и Detox
- **C)** Testcontainers и MockK
- **D)** Compose Test и XCUITest

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Таблица видов тестирования:

| Тип | Где выполняется | Инструменты |
|-----|-----------------|-------------|
| Unit | `commonTest` | JUnit 5, Kotest |
| Integration | `androidTest` / `iosTest` | Testcontainers, MockK |
| UI | Эмулятор/Симулятор | Compose Test, XCUITest |
| **E2E** | **Реальные устройства** | **Maestro, Detox** |

**Почему другие не подходят:**
- A — это unit-тестирование
- C — это интеграционные тесты
- D — это UI-тесты на эмуляторе/симуляторе

</details>

---

## Вопрос 2. Куда положить общие unit-тесты и Android UI-тесты соответственно?

- **A)** `commonTest` и `androidInstrumentedTest`
- **B)** `androidUnitTest` и `commonTest`
- **C)** `iosTest` и `androidUnitTest`
- **D)** Оба в `commonMain` — тесты компилируются вместе с кодом

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Структура из модуля:

```
shared/src/
├── commonMain/kotlin/                 # основной код
├── commonTest/kotlin/                 # общие unit-тесты
├── androidUnitTest/kotlin/            # Android unit-тесты (локальный JVM)
├── androidInstrumentedTest/kotlin/    # Android UI-тесты (устройство/эмулятор)
└── iosTest/kotlin/                    # iOS-тесты (XCTest)
```

UI-тесты Compose требуют рантайма Android, поэтому идут в `androidInstrumentedTest`.

**Почему другие не подходят:**
- B — `androidUnitTest` не является общим, а `commonTest` не содержит UI-тесты
- C — `iosTest` относится к iOS
- D — тесты всегда лежат в отдельных test source set'ах

</details>

---

## Вопрос 3. Какая библиотека для чего используется в зависимостях тестов?

- **A)** MockK — для тестирования Flow в `commonTest`; Turbine — только для JVM
- **B)** Turbine — для тестирования Flow в `commonTest`; MockK — для моков только на JVM (`androidUnitTest`); `kotlinx-coroutines-test` — для `runTest`
- **C)** JUnit 5 — только для iOS, XCTest — только для Android
- **D)** Все три библиотеки доступны во всех source set'ax без ограничений

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В `build.gradle.kts` модуля:

```kotlin
commonTestImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
commonTestImplementation("app.cash.turbine:turbine:1.0.0")
androidUnitTestImplementation("io.mockk:mockk:1.13.8")
```

Комментарий в коде прямо говорит: «MockK для моков (**только JVM**)». Turbine — «для тестирования Flow (от JetBrains)» — живёт в commonTest.

**Почему другие не подходят:**
- A — всё наоборот: Turbine в commonTest, MockK на JVM
- C — JUnit/XCTest распределены иначе (JUnit для Android/общих, XCTest для iOS)
- D — MockK недоступен в commonTest

</details>

---

## Вопрос 4. Что даёт `runTest` в этом тесте?

```kotlin
@Test
fun `adding habit updates UI state`() = runTest {
    val repository = HabitRepositoryMock()
    val viewModel = HabitViewModel(repository)
    viewModel.addHabit("Прогулка", HabitCategory.TRANSPORT)
    ...
}
```

- **A)** Запускает тест в отдельном потоке без корутин
- **B)** Создаёт `TestScope` с управляемым планировщиком и виртуальным временем, позволяя запускать `suspend`-код и ждать завершения корутин
- **C)** Автоматически создаёт мок репозитория
- **D)** Отключает все анимации в UI

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`runTest` из `kotlinx-coroutines-test` — стандартный способ тестировать корутины: он запускает тест в `TestScope`, подменяет `Dispatchers.Main` тестовым планировщиком, позволяет использовать виртуальное время (`advanceTimeBy`) и дожидаться завершения запущенных корутин. Без него `viewModelScope.launch { ... }` мог бы не успеть выполниться к моменту проверки.

**Почему другие не подходят:**
- A — речь именно о корутинах
- C — мок создаётся вручную (`HabitRepositoryMock()`)
- D — анимации здесь ни при чём

</details>

---

## Вопрос 5. Что делает эта конструкция с Turbine?

```kotlin
viewModel.uiState.test {
    val state = awaitItem()
    assertEquals(1, state.habits.size)
    assertEquals("Прогулка", state.habits[0].title)
}
```

- **A)** Подписывается на Flow `uiState`, ждёт следующее испущенное значение (`awaitItem`) и позволяет проверить его; по окончании тест сам проверит, что Flow завершён
- **B)** Вызывает `uiState` один раз как обычную функцию
- **C)** Создаёт новый `StateFlow` с начальным значением
- **D)** Запускает UI-тест на эмуляторе

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`Flow.test { ... }` из Turbine подписывается на поток и открывает «тестовую сессию». `awaitItem()` приостанавливает тест до очередного значения. Во втором тесте модуля показано, как после `awaitItem()` выполнить действие (`deleteHabit`) и снова вызвать `awaitItem()` — получить следующее состояние. В конце Turbine проверяет завершение потока (если поток должен завершиться, вызывают `awaitComplete()`).

**Почему другие не подходят:**
- B — `uiState` это Flow, а не функция
- C — тест не создаёт новый поток
- D — это unit-тест в `commonTest`, без эмулятора

</details>

---

## Вопрос 6. Почему в `commonTest` используется ручной `class HabitRepositoryMock : HabitRepository`, а не MockK?

- **A)** Потому что MockK работает только на JVM, а `commonTest` компилируется и для iOS
- **B)** Потому что MockK не умеет мокать интерфейсы
- **C)** Потому что `HabitRepository` — это класс, а не интерфейс
- **D)** Потому что Turbine запрещает использование любых моков

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В коде стоит комментарий: «Для commonTest используем простой mock». MockK подключён только как `androidUnitTestImplementation` и опирается на JVM-рефлексию, поэтому в common-тестах (которые запускаются и на iOS) его нельзя использовать. Ручная реализация интерфейса с `mutableListOf` — переносимый вариант. Для проверки ошибок в модуле есть вложенный `FailingRepository`, который всегда возвращает `Result.failure(Exception("Database error"))`.

**Почему другие не подходят:**
- B — MockK как раз умеет мокать интерфейсы
- C — `HabitRepository` — интерфейс
- D — Turbine и моки не конфликтуют

</details>

---

## Вопрос 7. Какой драйвер SQLDelight используется для тестирования Repository?

```kotlin
private val driver = JdbcSqliteDriver(JdbcConnectionConfig.inMemory)
private val database = EcoTrackDatabase(driver)
private val repository = HabitRepositoryImpl(database)
```

- **A)** `NativeSqliteDriver` с файлом на диске
- **B)** `JdbcSqliteDriver` с in-memory БД (`jdbc:sqlite::memory:`)
- **C)** `AndroidSqliteDriver` с `RoomDatabase`
- **D)** Реальный сервер PostgreSQL через Testcontainers

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В `HabitRepositoryImplTest` создаётся `JdbcSqliteDriver` с конфигурацией `JdbcConnectionConfig.inMemory`, где `url = "jdbc:sqlite::memory:"`. Это даёт изолированную БД в памяти: тесты не задевают реальные данные и не требуют эмулятора. Домашнее задание прямо предлагает «тесты для Repository с реальной БД (SQLDelight + JdbcDriver)».

**Почему другие не подходят:**
- A — нативный драйвер здесь не используется
- C — `AndroidSqliteDriver` и Room не упоминаются в модуле
- D — Testcontainers отнесён к интеграционным тестам в таблице, а не к этому примеру

</details>

---

## Вопрос 8. Найди ошибку в UI-тесте:

```kotlin
class HomeScreenTest {
    @get:Rule
    val activityRule = ActivityScenarioRule(MainActivity())

    @Test
    fun testHabitListDisplaysCorrectly() {
        activityRule.scenario.onActivity { activity ->
            val composeTestRule = createComposeRule()
            composeTestRule.onNodeWithText("Мои привычки").assertIsDisplayed()
        }
    }
}
```

- **A)** Ошибки нет — так и надо создавать Compose-правило внутри `onActivity`
- **B)** `createAndroidComposeRule<MainActivity>()` нужно объявить как `@get:Rule` на уровне класса и работать с ним напрямую; создание правила вручную внутри `onActivity` не подключает его к тесту
- **C)** Не хватает вызова `setContent {}` внутри `onActivity`
- **D)** `ActivityScenarioRule` нельзя использовать вместе с Compose

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B** (вопрос-ловушка!)

Внизу файла есть helper `fun createComposeRule() = createAndroidComposeRule<MainActivity>()`, но в тесте он вызывается **внутри** `onActivity`. Правило Compose должно быть полем класса с `@get:Rule`:

```kotlin
@get:Rule
val composeTestRule = createAndroidComposeRule<MainActivity>()
```

Тогда его matcher'ы (`onNodeWithText`, `onNodeWithContentDescription`) будут подаваться на уже запущенный Activity. Созданное же вручную правило не имеет смонтированного дерева Compose и не управляется JUnit.

**Почему другие не подходят:**
- A — это и есть ошибка примера
- C — `ActivityScenarioRule` уже запускает Activity с его `setContent`
- D — эти правила спокойно используются вместе, если объявлены правильно

</details>

---

## Вопрос 9. Какие API Compose Test используются в примерах модуля?

- **A)** `onNodeWithText`, `onNodeWithContentDescription`, `performClick`, `performTextInput`, `assertIsDisplayed`
- **B)** `findViewById`, `assertNotNull`, `TextView.getText()`
- **C)** `onNodeWithTag`, `awaitItem`, `test`
- **D)** `assertIsDisplayed` и `slideIntoContainer`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Примеры из `HomeScreenTest`:

```kotlin
composeTestRule.onNodeWithText("Мои привычки").assertIsDisplayed()
composeTestRule.onNodeWithContentDescription("Добавить привычку").performClick()
composeTestRule.onNodeWithText("Название").performTextInput("Прогулка")
composeTestRule.onNodeWithText("Прогулка").assertIsDisplayed()
```

`awaitItem`/`test` — из Turbine (unit-тесты), `slideIntoContainer` — из анимаций навигации Модуля 6.

**Почему другие не подходят:**
- B — это API классических Android View-тестов
- C — смешаны API разных модулей
- D — `slideIntoContainer` не относится к Compose Test

</details>

---

## Вопрос 10. Что делает `isMinifyEnabled = true` в `buildTypes.release` и откуда берутся пароли keystore?

- **A)** Включает R8 (уменьшение и обфускация кода); пароли берутся из `System.getenv("KEYSTORE_PASSWORD")` / `System.getenv("KEY_PASSWORD")`
- **B)** Включает только сжатие ресурсов; пароли хранятся в открытом виде в `build.gradle.kts`
- **C)** Отключает ProGuard; keystore создаётся автоматически при сборке
- **D)** Включает отладку по сети; пароли задаются в `local.properties`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле показаны два фрагмента:

```kotlin
release {
    isMinifyEnabled = true
    proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
}

signingConfigs {
    create("release") {
        storeFile = file("../keystore/ecotrack.keystore")
        storePassword = System.getenv("KEYSTORE_PASSWORD") ?: ""
        keyAlias = "ecotrack"
        keyPassword = System.getenv("KEY_PASSWORD") ?: ""
    }
}
```

Keystore создаётся командой `keytool -genkeypair` с `-keyalg RSA -keysize 2048 -validity 10000`. Пароли не должны лежать в репозитории — поэтому env-переменные.

**Почему другие не подходят:**
- B — minify это R8, а пароли не хардкодятся
- C — isMinifyEnabled включает, а не отключает оптимизацию
- D — `local.properties` в примере не используется

</details>

---

## Вопрос 11. Зачем в `proguard-rules.pro` написано `-keep class com.ecotrack.data.network.dto.** { *; }`?

- **A)** Чтобы удалить DTO из релизной сборки
- **B)** Чтобы сохранить классы DTO (от обфускации и вырезания) — они нужны сериализации Kotlinx
- **C)** Чтобы включить шифрование базы данных
- **D)** Чтобы заменить Ktor на OkHttp

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Файл правил модуля содержит:

```proguard
-keep class com.ecotrack.** { *; }
-keep class com.ecotrack.data.network.dto.** { *; }
-dontwarn kotlinx.serialization.**
-keep class app.cash.sqldelight.** { *; }
-dontwarn io.ktor.**
```

Сериализация обращается к классам и их полям по именам (в том числе через рефлексию/метаданные), поэтому R8 не должен их переименовывать или удалять. `-dontwarn` подавляет предупреждения от библиотек, чьи части безопасности не используются.

**Почему другие не подходят:**
- A — `-keep` делает противоположное
- C — шифрование БД здесь не настраивается
- D — правила не меняют HTTP-движок

</details>

---

## Вопрос 12. Зачем в `MainActivity.onCreate` перед инициализацией Koin стоит `delay(100)`?

```kotlin
lifecycleScope.launch {
    delay(100) // Даем время для отрисовки первого кадра
    setupKoin()
    setContent { EcoTrackTheme { /* ... */ } }
}
```

- **A)** Чтобы дождаться ответа сервера перед запуском UI
- **B)** Чтобы дать системе время отрисовать первый кадр и отложить тяжёлую инициализацию (Koin и другие зависимости), улучшив время запуска
- **C)** Чтобы загрузить шрифты из сети
- **D)** Чтобы обойти ошибку Composable-превью

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Комментарий в коде говорит сам за себя: «Даем время для отрисовки первого кадра», затем «Откладываем инициализацию тяжелых компонентов». Идея раздела «Оптимизация времени запуска» — не блокировать главный поток тяжёлой настройкой DI на старте. Критерий сдачи: время запуска < 3 секунд.

**Почему другие не подходят:**
- A — сетевые запросы здесь не выполняются
- C — шрифты упоминаются в дизайн-системе, но не здесь
- D — превью не связано с запуском приложения

</details>

---

## Вопрос 13. Какие инструменты модуль предлагает для профилирования Android и iOS?

- **A)** Android Studio Profiler (CPU, Memory, Network) и Xcode Instruments (Time Profiler, Allocations, Energy Log)
- **B)** Charles Proxy и mitmproxy
- **C)** Firebase Crashlytics и Sentry
- **D)** keytool и Transporter

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Раздел 4.3 перечисляет именно эти инструменты. Что проверять: утечки памяти в ViewModel и репозиториях, тяжёлые операции на main thread, избыточные сетевые запросы, частые wakeups и battery usage. Charles/mitmproxy — из Модуля 5 (инспекция HTTP), Crashlytics/Sentry — из Модуля 7 (краш-репорты), keytool/Transporter — из этого модуля, но для релиза, а не профилирования.

**Почему другие не подходят:**
- B — это перехват трафика, не профилировка
- C — это мониторинг ошибок, не профилировка
- D — это инструменты подписи и загрузки бинарника

</details>

---

## Вопрос 14. Что требуется для публикации в Google Play и App Store согласно модулю?

- **A)** В Google Play загружают APK, в App Store — AAB; достаточно скриншотов одного размера
- **B)** Google Play: **AAB (не APK)**, Target SDK 34+, политика конфиденциальности, скриншоты; App Store: iOS 15+, App Privacy Nutrition Label (обязательно с iOS 14+), бинарник через Xcode или Transporter
- **C)** Достаточно просто assemblyRelease — никаких дополнительных требований нет
- **D)** Обязательно использовать Firebase и Mixpanel, иначе ревью не пройти

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Требования из разделов 5.3 и 5.4:

| Google Play | App Store |
|-------------|-----------|
| AAB, не APK | Бинарник через Xcode/Transporter |
| Target SDK 34+ | iOS 15+ (рекомендуется) |
| Политика конфиденциальности | App Privacy Nutrition Label |
| Скриншоты всех размеров | Скриншоты iPhone/iPad |

Ревью: 3–7 дней в Google Play и 2–5 дней в App Store.

**Почему другие не подходят:**
- A — форматы перепутаны и скриншоты нужны для всех размеров
- C — требования обязательны
- D — аналитика не является требованием сторов

</details>

---

## Вопрос 15. Что верно про GitHub Actions workflow из модуля?

- **A)** Оба джоба запускаются на `ubuntu-latest` и собирают APK
- **B)** `build-android` идёт на `ubuntu-latest` (`./gradlew :shared:assembleRelease`, `:shared:testAndroidUnitTest`), а `build-ios` — на `macos-latest` с `:shared:assembleXCFramework`
- **C)** iOS-сборка запускается на Windows и использует Gradle wrapper без Xcode
- **D)** Workflow запускается только вручную, без триггеров `push` / `pull_request`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В `.github/workflows/build.yml`:

```yaml
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build-android:
    runs-on: ubuntu-latest
    # setup-java (JDK 17, temurin), chmod +x gradlew,
    # :shared:assembleRelease, :shared:testAndroidUnitTest, upload-artifact (APK)
  build-ios:
    runs-on: macos-latest
    # :shared:assembleXCFramework, upload-artifact (XCFramework)
```

iOS-таргету нужен macOS (Kotlin/Native и Xcode), поэтому runner другой. Артефакты загружаются через `actions/upload-artifact@v3`.

**Почему другие не подходят:**
- A — iOS-джоб требует macOS и XCFramework
- C — на Windows собрать iOS-фреймворк нельзя
- D — триггеры `push` и `pull_request` объявлены явно

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 14–15 | 🎉 Отлично! Тесты, оптимизация и релиз — пройдено. Уровень Middle KMP Developer твой |
| 11–13 | 📖 Перечитай разделы 2–4: Turbine, in-memory SQLDelight и ProGuard-правила |
| 8–10 | ⚠️ Пройди модуль ещё раз; сфокусируйся на source set'ax тестов и требованиях сторов |
| < 8 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🏁 Это финальный квиз курса. Возвращайся к предыдущим модулям, если ответы на вопросы давались с подглядыванием — повторение закрепляет материал лучше, чем спешка вперёд.

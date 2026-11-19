# 📝 Самопроверка: Модуль 10 — Тестирование и качество кода

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 11+ из 14 без подглядывания.

---

## Вопрос 1. Как распределены уровни в пирамиде тестирования из этого модуля?

- **A)** Unit — 10%, Integration — 30%, E2E — 20%, UI — 40%
- **B)** Unit — 40%, Integration — 30%, E2E — 20%, UI — 10%
- **C)** Unit, Integration, E2E и UI — по 25% каждый
- **D)** Unit — 50%, Integration — 20%, E2E — 20%, UI — 10%

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Модуль даёт такую пирамиду (от основания к вершине):

| Уровень | Доля | Характеристика |
|---------|:----:|----------------|
| Unit | 40% | очень быстрые (< 10 ms), дешёвые, без платформенного кода |
| Integration | 30% | быстрые (< 1 s), критичные пути |
| E2E | 20% | средние по скорости |
| UI | 10% | медленные (> 5 s), хрупкие, ломаются при изменении UI |

Чем выше уровень — тем дороже и медленнее тест, поэтому его меньше.

**Почему не другие:**
- A — перевёрнутая пирамида: самые дорогие UI-тесты стали бы самыми массовыми
- C — ровно 25% — это не пирамида, а «ромб»; модуль явно задаёт разные доли
- D — цифры не совпадают с приведённой схемой (40/30/20/10)

</details>

---

## Вопрос 2. Какую цель coverage ставит модуль и для какой части кода?

- **A)** 70%+ coverage для `commonMain` с помощью unit-тестов
- **B)** 80%+ coverage для Domain и Presentation слоёв
- **C)** 100% coverage всего кода, включая UI
- **D)** 50% coverage только для platform-specific кода

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле цель unit-тестов сформулирована прямо: **70%+ coverage для `commonMain`**. Та же цифра повторяется в домашнем задании и в критериях сдачи («70%+ coverage для commonMain (отчёт из IntelliJ/Xcode)»).

**Почему не другие:**
- B — это требование из модуля про архитектурные паттерны (там Domain + Presentation, 80%+), а не этого модуля
- C — 100% не требуется; модуль, наоборот, советует «не тестировать тривиальное» (геттеры/сеттеры)
- D — речь идёт именно об общей кроссплатформенной логике `commonMain`, а не о платформенной части

</details>

---

## Вопрос 3. Куда положить unit-тест бизнес-логики из `commonMain`?

- **A)** `androidUnitTest`
- **B)** `commonTest`
- **C)** `iosTest`
- **D)** `androidTest`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Примеры из модуля лежат в `shared/src/commonTest/kotlin/...`: `SkillsRepositoryTest.kt` и `SkillsViewModelTest.kt`. Именно `commonTest` компилируется для всех target'ов, поэтому тест общей логики выполняется один раз и проверяет тот код, который реально попадёт и в Android, и в iOS.

**Почему не другие:**
- A — `androidUnitTest` в модуле используется для integration-тестов SQLDelight (`DatabaseIntegrationTest.kt`)
- C — `iosTest` — для iOS-специфичных тестов (в примере — XCTest на Swift)
- D — `androidTest` — место Espresso UI-тестов, а не unit-тестов логики

</details>

---

## Вопрос 4. Разбери фрагмент unit-теста: что проверяет последняя строка?

```kotlin
@Test
fun `getSkills returns cached data when available`() = runTest {
    val expectedSkills = listOf(Skill(id = "1", name = "Kotlin"))
    every { localDataSource.getAllSkills() } returns expectedSkills

    val result = repository.getSkills()

    assertEquals(expectedSkills, result)
    verify(exactly = 0) { remoteDataSource.getSkills() }
}
```

- **A)** Что удалённый источник вызван ровно один раз
- **B)** Что сеть не вызвана вообще: данные взяты из непустого кэша
- **C)** Что кэш пуст и данные пришли с сервера
- **D)** Что `saveSkills` сохранёл результат в базу

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`SkillsRepository.getSkills()` устроен так: если `localDataSource.getAllSkills()` вернул непустой список — он сразу возвращается, и `remoteDataSource.getSkills()` **не вызывается**. Мок настроен на непустой кэш, значит `verify(exactly = 0)` фиксирует именно отсутствие сетевого запроса. Это и есть проверка поведения «cached data when available».

**Почему не другие:**
- A — `exactly = 1` появился бы в сценарии «cache is empty», где тест проверяет `verify(exactly = 1) { remoteDataSource.getSkills() }`
- C — для пустого кэша в примере пишут `every { localDataSource.getAllSkills() } returns emptyList()`
- D — сохранение проверяется отдельной строкой `verify(exactly = 1) { localDataSource.saveSkills(expectedSkills) }`, здесь её нет

</details>

---

## Вопрос 5. ❗Ловушка. Найди проблему в этом UI-тесте (код приведён в модуле как рабочий пример)

```kotlin
@Test
fun testSkillsListDisplaysItems() {
    ActivityScenario.launch(MainActivity::class.java)

    Thread.sleep(3000)

    onView(withText("Kotlin")).check(matches(isDisplayed()))
}
```

- **A)** Ничего не так — это эталонный пример из модуля, так и надо писать
- **B)** `Thread.sleep(3000)` — фиксированная задержка: тест медленный и хрупкий, лучше дождаться состояния (как `waitForExistence(timeout: 5)` в XCUITest-примере)
- **C)** Нельзя использовать `ActivityScenario` вместе с Espresso — нужно XCTest
- **D)** `onView(withText(...))` не работает в Espresso

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Это вопрос-ловушка: фрагмент действительно есть в модуле, но сам модуль называет UI-тесты **медленными и хрупкими** и советует «тестировать поведение, не реализацию». `Thread.sleep(3000)` — не проверка поведения, а слепая пауза: на быстром устройстве тест зря ждёт 3 секунды, на медленном — может упасть, если данные не успели загрузиться. В iOS-примере того же модуля используется корректный подход — `skillsQuery.waitForExistence(timeout: 5)`.

**Почему не другие:**
- A — «приведено в модуле» не значит «лучшая практика»; это рабочий, но компромиссный пример
- C — `ActivityScenario` + Espresso — стандартная связка для Android UI-тестов; XCTest — это iOS
- D — `onView(withText(...)).check(matches(isDisplayed()))` — как раз типичный Espresso-приём из примера

</details>

---

## Вопрос 6. Что проверяет этот тест на `StateFlow` и зачем здесь `.test {}`?

```kotlin
@Test
fun `isLoading is true during loading`() = runTest {
    every { repository.getSkills() } returns emptyList()
    val viewModel = SkillsViewModel(repository)

    viewModel.isLoading.test {
        assertEquals(false, awaitItem()) // Initial state
        viewModel.loadSkills()
        assertEquals(true, awaitItem())  // Loading started
        assertEquals(false, awaitItem()) // Loading finished
        cancelAndIgnoreRemainingEvents()
    }
}
```

- **A)** Последовательность состояний `false → true → false`, собранную с помощью Turbine (`import app.cash.turbine.test`)
- **B)** Только конечное значение `isLoading`, а `.test {}` — аналог `assertEquals`
- **C)** Что `isLoading` — обычный `List<Boolean>`, и `.test {}` его сортирует
- **D)** Что корутии вовсе не запускаются, поэтому 💡 `cancelAndIgnoreRemainingEvents()` обязателен

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В импортах теста стоит `app.cash.turbine.test` — это Turbine, утилита для тестирования Flow. `awaitItem()` по очереди читает испускаемые значения: стартовое `false` (StateFlow создан с `MutableStateFlow(false)`), затем `true` (в `loadSkills()` выставлен `_isLoading.value = true`) и снова `false` (в `finally`). `cancelAndIgnoreRemainingEvents()` завершает сборку. `runTest` создаёт тестовый корутин-диспетчер.

**Почему не другие:**
- B — Turbine проверяет именно **последовательность**, а не только последнее значение
- C — `isLoading` имеет тип `StateFlow<Boolean>`, это не `List`
- D — корутины работают: `viewModel.loadSkills()` запускает `suspend fun`; `cancelAndIgnoreRemainingEvents()` лишь закрывает тест Flow

</details>

---

## Вопрос 7. Где лежит integration-тест SQLDelight и как в нём создаётся драйвер?

- **A)** В `commonTest`, драйвер создаётся через `Room.databaseBuilder`
- **B)** В `androidUnitTest`; `TestSqlDriverFactory.createDriver()` создаёт `TestSqlDriver(Schema, "test.db")`, а в `@After` вызывается `driver.close()`
- **C)** В `iosTest` на Swift через XCTestCase
- **D)** В `androidTest`, через Espresso `ActivityScenario`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Файл из модуля — `shared/src/androidUnitTest/kotlin/data/DatabaseIntegrationTest.kt`. В `@Before` он создаёт in-memory базу: `driver = TestSqlDriverFactory.createDriver()`, `database = AppDatabase(driver)`; `TestSqlDriverFactory` использует `TestSqlDriver(schema = com.skillsync.database.Schema, name = "test.db")`. В `@After` драйвер закрывается. Это integration-уровень: проверяется взаимодействие Repository/запросов с реальной БД.

**Почему не другие:**
- A — `commonTest` в модуле отведён под unit-тесты; Room здесь не используется, пример использует SQLDelight `SqlDriver`
- C — XCTest в модуле проверяет `SkillsRepository` с моками, а не SQLDelight
- D — Espresso — это UI-тесты (`androidTest`), не работа с БД

</details>

---

## Вопрос 8. Разбери XCTest-фрагмент: что проверяется?

```swift
localDataSource.mockedGetAllSkills = { [] }
remoteDataSource.mockedGetSkills = { expectedSkills }

let result = await repository.getSkills()

XCTAssertEqual(expectedSkills, result)
XCTAssertTrue(remoteDataSource.getSkillsCalled == true)
```

- **A)** Что данные взяты из кэша, а сеть не вызывалась
- **B)** Что пустой кэш привёл к запросу в сеть, и флаг вызова выставлен
- **C)** Что Repository вернул ошибку и тест это фиксирует
- **D)** Что `getSkills()` синхронный и `await` здесь лишний

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Мок локального источника возвращает `[]`, значит Repository идёт в `remoteDataSource.getSkills()` (та же логика cache-then-network, что и в Kotlin-тесте). `MockSkillsRemoteDataSource.getSkills()` выставляет `getSkillsCalled = true`, поэтому вторая проверка подтверждает обращение к сети. `@testable import shared` подключает KMP-фреймворк.

**Почему не другие:**
- A — для кэш-сценария в модуле пишут `mockedGetAllSkills = { expectedSkills }` и проверяют `getSkillsCalled == false`
- C — тест сравнивает результат с ожидаемым списком, это сценарий успеха
- D — `getSkills()` — `suspend fun`, поэтому в Swift он вызывается через `await`

</details>

---

## Вопрос 9. Какие инструменты и сценарии используются для UI-тестов на двух платформах?

- **A)** Android — Espresso (`androidTest`), iOS — XCUITest; сценарии: список навыков, кнопка «Add Skill», поиск
- **B)** Android — XCTest, iOS — Espresso; сценарии те же
- **C)** Обе платформы — Detekt, проверяющий UI-разметку
- **D)** Только XCUITest, Android-приложение тестируется unit-тестами

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле приведены `SkillsScreenTest.kt` (Espresso: `ActivityScenario.launch`, `onView`, `withId(R.id.add_skill_button)`) и `SkillsScreenTests.swift` (XCUITest: `XCUIApplication`, `app.buttons["Add Skill"]`, `app.textFields["Search"]`). Оба файла проверяют одинаковые три сценария: отображение списка, открытие диалога добавления и фильтрацию поиском.

**Почему не другие:**
- B — инструменты перепутаны: XCTest/iOS и Espresso/Android
- C — Detekt — Kotlin-линтер для code quality, не UI-тестирование
- D — для Android в модуле явно есть отдельный Espresso-набор

</details>

---

## Вопрос 10. Почему iOS-джоб в GitHub Actions использует `macos-latest` и какую команду запускает?

- **A)** Потому что Kotlin/Native-тесты iOS-таргета требуют macOS/Xcode; команда `./gradlew :shared:iosSimulatorArm64Test`
- **B)** Потому что Gradle не работает на Linux; команда `./gradlew :shared:test`
- **C)** Потому что Detekt работает только на macOS; команда `./gradlew :shared:detekt`
- **D)** Потому что iOS-тесты — это Espresso; команда `./gradlew connectedCheck`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В `ci.yml` джоб `test-ios` объявлен с `runs-on: macos-latest` и выполняет `./gradlew :shared:iosSimulatorArm64Test`, а затем `./gradlew :apps:skillsync-ios:assembleDebug`. Остальные джобы (`test-common`, `test-android`, `code-quality`) идут на `ubuntu-latest`. Для симулятора Apple Silicon нужен соответствующий target — `iosSimulatorArm64Test`.

**Почему не другие:**
- B — `test-common` спокойно выполняется на ubuntu (`./gradlew :shared:test`)
- C — Detekt в модуле запускается в джобе `code-quality` на `ubuntu-latest`
- D — `connectedCheck` — это Android UI-тесты в джобе `test-android`, и для них нужен эмулятор/устройство

</details>

---

## Вопрос 11. Как связаны джобы `build-release` и `deploy` в CI-пайплайне?

- **A)** `build-release` ждёт все четыре проверки (`needs: [test-common, test-android, test-ios, code-quality]`) и идёт только на `main`; `deploy` ждёт `build-release` и тоже только на `main`
- **B)** `deploy` запускается сразу после `checkout`, чтобы быстрее выкатить изменения
- **C)** `build-release` запускается на каждый pull request в `main`
- **D)** `code-quality` запускается после `deploy`, чтобы не тормозить релиз

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле:

```yaml
build-release:
  needs: [test-common, test-android, test-ios, code-quality]
  if: github.ref == 'refs/heads/main'
deploy:
  needs: build-release
  if: github.ref == 'refs/heads/main'
```

То есть релизная сборка возможна только когда зелёны все тесты и проверки качества, и только в ветке `main`. Деплой — следующий за ней шаг. Это и есть защита магистрали от сломанных релизов.

**Почему не другие:**
- B — деплой без проверок означал бы выкладку непроверенного кода
- C — условия `if: github.ref == 'refs/heads/main'` нет ни у PR-триггера, ни у сборки релиза
- D — `code-quality` — одна из четырёх зависимостей `build-release`, а не шаг после деплоя

</details>

---

## Вопрос 12. Что означают настройки `detekt-config.yml` из модуля?

```yaml
config:
  builds:
    excludedPaths:
      - '.*Test\.kt'
style:
  MagicNumber:
    active: false # Отключаем для KMP (константы платформ)
  WildcardImport:
    active: true
```

- **A)** Тестовые файлы исключены из проверки; `MagicNumber` выключен для KMP; `WildcardImport` включён
- **B)** Detekt проверяет только тесты; `MagicNumber` включён; `WildcardImport` выключен
- **C)** Все проверки выключены, Detekt в CI не запускается
- **D)** `excludedPaths` исключает продакшн-код, `MagicNumber` проверяет только тесты

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`excludedPaths: '.*Test\.kt'` убирает из анализа файлы тестов. `MagicNumber: active: false` — комментарий в модуле объясняет: правило отключается для KMP, потому что встречаются константы платформ. `WildcardImport: active: true` оставляет запрет звёздных импортов (с исключением `kotlinx.*`). В CI джоб `code-quality` запускает `./gradlew :shared:detekt` и `./gradlew :shared:ktlintCheck`.

**Почему не другие:**
- B — всё наоборот: тесты исключены, `MagicNumber` выключен, `WildcardImport` включён
- C —активны `processors`, `console-reports`, `complexity`, `style`, `coroutines` и отдельные правила
- D — регулярное выражение явно указывает на `Test.kt`, то есть исключаются тесты, не продакшн-код

</details>

---

## Вопрос 13. Какой набор советов модуля правильно описывает хороший тест?

- **A)** Начинать с unit-тестов, тестировать поведение, а не реализацию, использовать Arrange–Act–Assert, не тестировать тривиальные геттеры/сеттеры, автоматизировать всё через CI
- **B)** Сначала писать UI-тесты, затем unit; тестировать приватные методы напрямую; геттеры покрывать обязательно
- **C)** Тесты должны проверять внутреннюю реализацию и каждый `if` в коде
- **D)** CI запускать вручную по запросу, чтобы не расходовать ресурсы

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Это почти дословно раздел «Советы по выполнению»: 1) начни с unit-тестов (быстрые и дешёвые); 2) тестируй поведение, не реализацию; 3) Arrange–Act–Assert — те примеры (`// Arrange`, `// Act`, `// Assert`) есть в каждом тесте Repository; 4) не тестируй тривиальное; 5) автоматизируй всё — CI/CD запускает тесты при push/PR.

**Почему не другие:**
- B — противоречит совету «начните с unit-тестов» и «не тестируйте тривиальное»
- C — проверка реализации делает тест хрупким; модуль рекомендует проверять поведение
- D — модуль требует автоматического запуска тестов при push/PR, а не ручного

</details>

---

## Вопрос 14. Итоговый: что из этого полностью удовлетворяет критериям сдачи домашнего задания?

- **A)** Unit-тесты Repository/ViewModel/Use Cases в `commonTest` с MockK и 70%+ coverage; SQLDelight-тесты в `androidUnitTest` и XCTest на iOS; Espresso и XCUITest на одних сценариях; GitHub Actions запускает всё автоматически, плюс Detekt
- **B)** Только UI-тесты Espresso на трёх сценариях — они проверяют приложение «как пользователь»
- **C)** Unit-тесты только в iOS-проекте на XCTest плюс SwiftLint
- **D)** Unit-тесты на все геттеры `data class Skill` и ручной прогон тестов перед релизом

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Критерии сдачи в модуле: 70%+ coverage для `commonMain`; все unit-тесты проходят в CI; integration-тесты покрывают работу с БД; UI-тесты проходят на эмуляторах/симуляторах; CI-пайплайн настроен и работает. Плюс раздел Code Quality требует Detekt (и SwiftLint для iOS, если есть). Вариант A закрывает все пункты и использует правильные source set'ы и инструменты.

**Почему не другие:**
- B — нет unit и integration-уровней, нет CI и покрытия
- C — тестируется только iOS; общая логика `commonMain` остаётся непроверенной на Android
- D — модуль прямо советует не тестировать тривиальное, а ручной прогон противоречит требованию автоматизации

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 12–14 | 🎉 Отлично! Пирамида тестирования, CI и код-качество — твои |
| 9–11 | 📖 Перечитай секции про `commonTest`/MockK и про GitHub Actions; посмотри вопросы, где ошибся |
| 5–8 | ⚠️ Пройди модуль ещё раз: разбери разницу между unit/integration/UI и их source set'ами |
| < 5 | 🆘 Начни с начала: пирамида тестирования и Arrange–Act–Assert, потом вернись к квизу |

---

> 🔜 Дальше — модуль архитектурных паттернов: там те же слои, но уже с Clean Architecture, MVI и Koin.

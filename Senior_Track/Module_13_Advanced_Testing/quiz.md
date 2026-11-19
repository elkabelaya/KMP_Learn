# 📝 Самопроверка: Модуль 13 — Advanced Testing Strategies

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 10+ из 14 без подглядывания.

---

## Вопрос 1. Где объявляются `kotlin("test")`, MockK, Turbine и `kotlinx-coroutines-test`?

- **A)** В `androidUnitTest`
- **B)** В `commonTest`
- **C)** В `iosTest`
- **D)** Отдельно в каждом платформенном source set'е

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В конфигурации модуля все четыре зависимости лежат в `commonTest`:

```kotlin
val commonTest by getting {
    dependencies {
        implementation(kotlin("test"))
        implementation("io.mockk:mockk:1.13.4")
        implementation("app.cash.turbine:turbine:1.0.0")
        implementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
    }
}
```

Так тесты и их утилиты доступны для всех target'ов. В `androidUnitTest` лежат только `kotlin("test-junit")` и `junit:junit:4.13.2`, в `iosTest` — только `kotlin("test")`.

**Почему другие не подходят:**
- A — туда добавлены лишь JUnit-зависимости для Android
- C — iOS-набор содержит только Kotlin Test
- D — дублирование; общие зависимости вынесены в `commonTest`

</details>

---

## Вопрос 2. Какая библиотека в конфигурации модуля предназначена для тестирования Flow?

- **A)** MockK (`io.mockk:mockk:1.13.4`)
- **B)** Turbine (`app.cash.turbine:turbine:1.0.0`)
- **C)** JaCoCo
- **D)** JUnit 4 (`junit:junit:4.13.2`)

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Turbine — стандартный инструмент для тестирования `Flow`/`StateFlow`: он позволяет ждать события (`awaitItem()`, `awaitComplete()`, `awaitError()`) в детерминированном виде. В модуле он подключён в `commonTest` рядом с `kotlinx-coroutines-test`.

**Почему другие не подходят:**
- A — MockK создаёт моки и проверяет вызовы (`every`, `verify`), но не «слушает» Flow
- C — JaCoCo считает покрытие, а не тестирует потоки
- D — JUnit — это раннер/ассерты, для Flow он ничего не знает

</details>

---

## Вопрос 3. Разбери фрагмент теста use case:

```kotlin
@Test
fun `returns error when repository throws exception`() = runTest {
    val expectedError = Exception("Database error")
    every { mockRepository.getAllSkills() } throws expectedError

    val result = useCase()

    assertTrue(result.isFailure)
    assertEquals(expectedError, result.exceptionOrNull())
}
```

Почему `result.isFailure == true`?

- **A)** `runTest` автоматически превращает все исключения в `Result.failure`
- **B)** MockK заставляет `getAllSkills()` бросить исключение, а `GetSkillsUseCase` ловит его в `try/catch` и возвращает `Result.failure(e)`
- **C)** Неявно вызывается `assertThrows`, который и меняет результат
- **D)** Любая `suspend`-функция в Kotlin всегда возвращает `Result`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Классический Arrange–Act–Assert:

1. **Arrange:** `every { ... } throws expectedError` настраивает мок так, что репозиторий бросает исключение.
2. **Act:** `useCase()` внутри себя вызывает репозиторий в `try { ... } catch (e: Exception) { Result.failure(e) }`.
3. **Assert:** проверяем `isFailure` и, что важно, **тот же самый объект ошибки** через `exceptionOrNull()`.

`Result` здесь — результат явной обработки ошибки в use case, а не магия тестового раннера.

**Почему другие не подходят:**
- A — `runTest` лишь управляет корутинами и виртуальным временем
- C — `assertThrows` здесь не вызывается
- D — `suspend` ничего не говорит о возвращаемом типе

</details>

---

## Вопрос 4. Тест ViewModel: после `every { mockUseCase() } returns Result.success(expectedSkills)` и `advanceUntilIdle()` чему равны поля состояния?

```kotlin
val state = viewModel.state.value
```

- **A)** `isLoading = true`, `skills = expectedSkills`, `error = null`
- **B)** `isLoading = false`, `skills = expectedSkills`, `error = null`
- **C)** `isLoading = false`, `skills = []`, `error = "Network error"`
- **D)** `isLoading = true`, `skills = []`, `error = null`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В `init` ViewModel запускает `loadSkills()`: сначала выставляется `isLoading = true`, затем в ветке `is Result.Success` состояние обновляется на `skills = result.data, isLoading = false`, а `error` остаётся `null`. `advanceUntilIdle()` (из `kotlinx-coroutines-test`) прокручивает виртуальное время, чтобы дождаться завершения корутины перед проверкой.

Именно поэтому отдельный тест `state shows loading initially` проверяет `true` **до** продвижения времени, а этот — `false` **после**.

**Почему другие не подходят:**
- A — это промежуточное состояние, которое уже сменилось к моменту проверки
- C — ошибки не было, мок вернул `Success`
- D — игнорирует успешную загрузку после `advanceUntilIdle()`

</details>

---

## Вопрос 5. 🎯 Вопрос-ловушка: что сделает этот extension из модуля?

```kotlin
// Test extension for StateFlow
fun <T> StateFlow<T>.test(): Turbine<T> = this.test()
```

- **A)** Корректно делегирует вызов Turbine — так и надо тестировать `StateFlow`
- **B)** Не скомпилируется: Turbine не умеет работать с `StateFlow`
- **C)** Рекурсивно вызовет сам себя → `StackOverflowError`
- **D)** Вернёт `Turbine`, но только после вызова `advanceUntilIdle()`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

Выглядит как безобидная обёртка, но внутри тела `this.test()` разрешается **в тот же самый extension** (он ближе по типу ресивера, чем `Flow<T>.test()` из Turbine). Получается бесконечная рекурсия.

Правильные варианты: вызывать `Flow<T>.test()` из Turbine напрямую (`flow.test { ... }`) либо дать собственному extension другое имя. Та же ошибка есть и в `FlowExtensions.kt` модуля:

```kotlin
fun <T> Flow<T>.debounce(delayMillis: Long): Flow<T> = this.debounce(delayMillis)
```

Здесь `this.debounce(...)` тоже замыкается на собственную функцию, а не на стандартный оператор `kotlinx.coroutines.flow.debounce`.

**Почему другие не подходят:**
- A — именно это и есть ловушка: «делегирования» на самом деле нет
- B — Turbine как раз работает с `Flow`, проблема в разрешении имени
- D — время здесь ни при чём, падение произойдёт до любой проверки

</details>

---

## Вопрос 6. Чем `FakeSkillsRepository` отличается от `mock<SkillsRepository>()`?

- **A)** Fake — рабочая in-memory реализация на `MutableList<Skill>` с реальными CRUD-методами; мок MockK — стаб с заданными ответами и проверкой вызовов через `verify`
- **B)** Fake автоматически генерируется MockK, а мок приходится писать руками
- **C)** Fake нужен только для unit-тестов use case, а мок — только для integration-тестов
- **D)** Разницы нет, это два названия одного и того же

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле чёткое разделение:

| Инструмент | Где применяется | Суть |
|------------|-----------------|------|
| `mock<SkillsRepository>()` | unit-тесты use case / ViewModel | `every { ... } returns/throws`, `verify { ... }` |
| `FakeSkillsRepository` | integration-тесты | хранит `mutableListOf`, реально фильтрует, добавляет, обновляет и удаляет |

Fake позволяет проверять **поведение системы как целого** (после `addSkill` в репозитории действительно 4 элемента), а мок — изолировать юнит и проверить контракт взаимодействий.

**Почему другие не подходят:**
- B — всё наоборот, Fake пишется вручную
- C — как раз наоборот: мок для изолированных unit-тестов, Fake для интеграционных
- D — это разные виды test doubles

</details>

---

## Вопрос 7. Что вернёт `updateSkill` для несуществующего id?

```kotlin
override suspend fun updateSkill(skill: Skill): Result<Unit> {
    val index = skills.indexOfFirst { it.id == skill.id }
    if (index != -1) {
        skills[index] = skill
        return Result.success(Unit)
    }
    return Result.failure(Exception("Skill not found"))
}
```

- **A)** `Result.success(Unit)` — элемент будет добавлен в конец списка
- **B)** `Result.failure(Exception("Skill not found"))`, список не изменится
- **C)** `null`
- **D)** `IllegalArgumentException` будет выброшен наружу

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`indexOfFirst` возвращает `-1`, если совпадений нет. Условие `index != -1` не выполняется, поэтому метод возвращает `Result.failure(Exception("Skill not found"))` — **не выбрасывая** исключение. Аналогично устроен `deleteSkill`: `removeAll` возвращает `false`, и метод тоже отдаёт `Result.failure`.

Обрати внимание: в отличие от `updateSkill`, `addSkill` просто добавляет элемент и всегда возвращает `Result.success(Unit)`.

**Почему другие не подходят:**
- A — обновление существующего элемента — это ветка `index != -1`
- C — метод возвращает `Result`, а не nullable значение
- D — исключение «упаковано» в `Result`, наружу оно не уйдёт

</details>

---

## Вопрос 8. В `setup()` интеграционного теста созданы 3 навыка: Kotlin (programming), Swift (programming), Figma (design). Чему равно `getSkillsByCategory("programming").size`?

- **A)** 0
- **B)** 1
- **C)** 2
- **D)** 3

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

`FakeSkillsRepository.getSkillsByCategory` делает `skills.filter { it.categoryId == categoryId }`. Под «programming» подходят Kotlin и Swift — итого 2. В тесте это дополнительно проверяется через `assertTrue(result.all { it.categoryId == "programming" })`.

Для сравнения: `getAllSkills()` вернёт 3, а после `addSkill(...)` — уже 4.

**Почему другие не подходят:**
- A — фильтр работает, репозиторий не пустой
- B — забыт один из двух навыков категории «programming»
- D — это размер всего списка, включая Figma

</details>

---

## Вопрос 9. Какие rule и matcher используются в Android Compose UI-тесте для проверки индикатора загрузки?

- **A)** `createAndroidComposeRule()` и `onNodeWithText("Loading")`
- **B)** `createComposeRule()` и `onNodeWithContentDescription("Loading")`
- **C)** `createComposeRule()` и `onNodeWithTag("loading")`
- **D)** `ActivityScenarioRule` и `onNodeWithText("Loading")`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В модуле объявлен `@get:Rule val composeTestRule = createComposeRule()`, а индикатор ищется по content description:

```kotlin
composeTestRule.onNodeWithContentDescription("Loading").assertIsDisplayed()
```

Так же проверяется кнопка обновления — `onNodeWithContentDescription("Refresh").performClick()`. Для текста навыков используется `onNodeWithText("Kotlin").assertIsDisplayed()`.

**Почему другие не подходят:**
- A — в примере другой rule, и иконка не имеет текста «Loading»
- C — тег `testTag` здесь не используется
- D — `ActivityScenarioRule` не работает с Compose-деревом напрямую

</details>

---

## Вопрос 10. Зачем в UI-тестах объявлен `expect fun createUiTestRunner(): UiTestRunner`?

- **A)** Чтобы общая логика теста (`launchApp`, `tapOn`, `assertElementVisible`) работала с разными платформенными раннерами: `AndroidUiTestRunner` и `IosUiTestRunner`
- **B)** Чтобы полностью заменить XCUITest на Kotlin-код без платформенных реализаций
- **C)** Чтобы тесты Compose можно было запускать без эмулятора
- **D)** Это способ подключить MockK к UI-тестам

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Интерфейс `UiTestRunner` задаёт unify-API (`launchApp()`, `tapOn(element)`, `assertElementVisible(element)`), а `expect/actual` подставляет конкретную реализацию для Android и iOS. Благодаря этому сценарий `SkillsUiTest` пишется один раз:

```kotlin
testRunner.launchApp()
testRunner.tapOn("Skills")
testRunner.assertElementVisible("Kotlin")
```

Это тот же принцип, что и в основном коде: общая логика в common, платформенные детали — в actual.

**Почему другие не подходят:**
- B — без actual-реализаций тест не соберётся; XCUITest никто не отменял
- C — эмулятор/симулятор всё равно нужен
- D — MockK здесь не участвует

</details>

---

## Вопрос 11. Сколько `actual`-реализаций нужно для `expect fun createUiTestRunner()` и где?

```kotlin
// obщая часть
expect fun createUiTestRunner(): UiTestRunner

// androidTest
actual fun createUiTestRunner(): UiTestRunner = AndroidUiTestRunner()
// iosTest
actual fun createUiTestRunner(): UiTestRunner = IosUiTestRunner()
```

- **A)** Одна — в `commonTest`
- **B)** Две — в `androidTest` и `iosTest`
- **C)** Три — в `androidTest`, `iosTest` и `commonTest`
- **D)** Ни одной, достаточно интерфейса `UiTestRunner`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Правило то же, что для обычного `expect/actual`: по одной реализации на каждый source set, где реально запускаются тесты. Здесь это `androidTest` → `AndroidUiTestRunner()` и `iosTest` → `IosUiTestRunner()`. В `commonTest` лежит только объявление `expect` и общая логика сценария.

**Почему другие не подходят:**
- A — одна реализация не покроет вторую платформу, iOS-тест не соберётся
- C — в `commonTest` actual быть не должно, там общий контракт
- D — интерфейс описывает поведение, но не говорит, какой раннер создать

</details>

---

## Вопрос 12. Что делает блок в `android/build.gradle.kts`?

```kotlin
tasks.withType<Test> {
    isIgnoreFailures = true
    finalizedBy("createJacocoReport")
}
```

- **A)** Заставляет все тесты всегда проходить
- **B)** Гарантирует, что отчёт JaCoCo будет сформирован после тестовых задач, даже если тесты упали
- **C)** Запускает генерацию отчёта до выполнения тестов
- **D)** Включает покрытие только для `iosTest`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`finalizedBy("createJacocoReport")` привязывает задачу отчёта к каждой задаче типа `Test`: она выполнится после неё независимо от результата. А `isIgnoreFailures = true` не даёт падению тестов прервать сборку до генерации отчёта — так ты видишь и сами тесты, и цифры покрытия.

Сам `JacocoReport` берёт классы из `build/intermediates/javac/debug/classes` и `build/tmp/kotlin-classes/debug`, а execution data — из `build/jacoco`.

**Почему другие не подходят:**
- A — тесты по-прежнему могут падать, меняется только обработка результата
- C — `finalizedBy` — это «после», а не «до»
- D — конфигурация относится к Android-модулю, к iOS отношения не имеет

</details>

---

## Вопрос 13. Что делает `scripts/generate-coverage-report.sh` из модуля?

- **A)** Удаляет старые отчёты, запускает `./gradlew testDebugUnitTest createJacocoReport --continue` и открывает HTML-отчёт в браузере
- **B)** Только открывает уже существующий отчёт
- **C)** Запускает все тесты на Android, iOS и Desktop и публикует отчёт на CI
- **D)** Считает покрытие и автоматически исправляет непокрытые ветки

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Скрипт последовательно:

1. `rm -rf build/reports/jacoco` — чистит предыдущий отчёт;
2. `./gradlew testDebugUnitTest createJacocoReport --continue` — прогоняет тесты и строит отчёт, `--continue` позволяет дойти до генерации отчёта даже при падении задач;
3. `open build/reports/jacoco/html/index.html` на macOS или `xdg-open` на Linux.

**Почему другие не подходят:**
- B — он и запускает тесты, и чистит каталог
- C — речь только про Android `testDebugUnitTest`
- D — скрипт лишь измеряет покрытие, а не меняет код

</details>

---

## Вопрос 14. В требованиях ДЗ — «минимум 80% coverage» для unit-тестов, в критериях сдачи — «>70% coverage». Как это правильно понять?

- **A)** Это противоречие: модуль нужно исправить
- **B)** 80% — цель при написании unit-тестов, а >70% — минимальный порог итогового отчёта, который проверяет преподаватель
- **C)** Достаточно 70%, требование 80% можно игнорировать
- **D)** Покрытие в критериях сдачи не упоминается

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Требования задают, к чему стоит стремиться при реализации (unit tests для всех use cases, минимум 80% coverage), а критерии сдачи фиксируют проходной порог для итоговой работы (coverage report показывает >70% coverage). Это разные уровни ожиданий: цель выше проходной планки — нормальная практика.

Остальные критерии сдачи: unit tests для 100% use cases, integration tests с fake implementations, UI tests для 3+ основных экранов.

**Почему другие не подходят:**
- A — это не ошибка документа, а два разных уровня требований
- C — игнорировать заявленную цель — риск не дотянуть до порога
- D — порог >70% прямо указан в критериях

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 12–14 | 🎉 Отлично! Тестовая инфраструктура KMP у тебя в голове: можно смело идти дальше |
| 9–11 | 📖 Перечитай секции 2–4, особенно разницу между mock и Fake и `expect/actual` в UI-тестах |
| 5–8 | ⚠️ Пройди модуль ещё раз и перепиши один тест use case и один integration-тест своими руками |
| < 5 | 🆘 Давай разберём вместе — начни с `commonTest`, `runTest` и JaCoCo, потом вернись к квизу |

---

> 🔜 Следующий модуль Senior Track — продвинутые темы: C++ interop, ML ивенты и сложная конкурентность. Там тоже будут ловушки в коде — будь внимателен. 🚀

# 📝 Самопроверка: Модуль 4 — AI-ассистированное тестирование

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 11+ из 15 без подглядывания.

---

## Вопрос 1. В какой роли выступает AI в шаблоне промпта для генерации unit-тестов?

- **A)** Senior QA Engineer, эксперт в тестировании Kotlin-кода
- **B)** Principal Architect
- **C)** UI/UX дизайнер
- **D)** DevOps-инженер, настраивающий CI

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаблон начинается: *«Ты — Senior QA Engineer, эксперт в тестировании Kotlin кода»*, после чего следует задача написать исчерпывающие unit-тесты, а требования: JUnit 5 + AssertJ (или Kotlin Test), happy path/edge cases/error scenarios, моки через MockK, минимум 90% строк кода.

**Почему другие не подходят:**
- B — роль архитектора используется в Модуле 3
- C, D — дизайн и CI не входят в шаблон генерации тестов

</details>

---

## Вопрос 2. Что НЕ входит в требования шаблона unit-тестов?

- **A)** Покрыть минимум 90% строк кода
- **B)** Создавать mock-объекты для всех зависимостей через MockK
- **C)** Структура Arrange → Act → Assert
- **D)** Использовать Mockito вместо MockK

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: D**

В шаблоне прямо указаны JUnit 5 + AssertJ (или Kotlin Test) и MockK для всех зависимостей. Mockito в модуле не упоминается; для Kotlin Multiplatform рекомендуется MockK.

**Почему другие не подходят:**
- A — требование «покрывай минимум 90% строк кода» есть в списке
- B — отдельный пункт требований
- C — Arrange/Act/Assert задаётся разделом «Структура тестов»

</details>

---

## Вопрос 3. Какому шаблону именования тестов учит модуль?

- **A)** `test1`, `test2`, `test3`
- **B)** `should_[expectedBehavior]_when_[condition]`
- **C)** `[className]_Test_Function`
- **D)** `given_[input]_then_[output]`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В шаблоне: *«Назови тесты описательно: `should_[expectedBehavior]_when_[condition]`»*. Примеры из кода: ``should complete habit and increment streak when habit exists``, ``should return HabitNotFoundError when habit does not exist``, ``should return RepositoryError when save fails``.

**Почему другие не подходят:**
- A — неинформативные имена
- C, D — другие соглашения, в модуле не используются

</details>

---

## Вопрос 4. Какие зависимости нужно замокить для `CompleteHabitUseCase`?

```kotlin
class CompleteHabitUseCase(
    private val habitRepository: HabitRepository,
    private val dateProvider: DateProvider
) {
    suspend operator fun invoke(habitId: String): Result<Habit> { /* ... */ }
}
```

- **A)** Только `habitRepository`
- **B)** Только `dateProvider`
- **C)** `habitRepository` и `dateProvider`
- **D)** Ни одну — используем реальные реализации

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

В тесте создаются два мока: `private val habitRepository = mock<HabitRepository>()` и `private val dateProvider = mock<DateProvider>()`, после чего в `@BeforeTest fun setup()` они передаются в конструктор `CompleteHabitUseCase`. Требование шаблона — моки для **всех** зависимостей.

**Почему другие не подходят:**
- A, B — неполный набор зависимостей
- D — реальный репозиторий и провайдер даты сделали бы тест медленным и недетерминированным

</details>

---

## Вопрос 5. Какое изменение streak ожидается в тесте успешного завершения привычки?

```kotlin
val existingHabit = Habit(id = habitId, title = "Test Habit", streak = 5, isCompleted = false)
val completedHabit = existingHabit.copy(isCompleted = true, streak = 6)
```

- **A)** 5 → 5, меняется только `isCompleted`
- **B)** 5 → 6, `isCompleted = true`
- **C)** 5 → 0 (сброс streak)
- **D)** 5 → 10 (удвоение)

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В тесте ``should complete habit and increment streak when habit exists`` проверяется `assertThat(habit.streak).isEqualTo(6)` и `assertThat(habit.isCompleted).isTrue()`. Ожидаемый объект `completedHabit` передаётся в `saveHabit`, а вызов дополнительно проверяется через `verify`.

**Почему другие не подходят:**
- A — логика завершения привычки должна увеличивать серию
- C — сброс streak противоречит названию теста
- D — в коде используется +1, а не удвоение

</details>

---

## Вопрос 6. Как в тесте проверяется, что `saveHabit` действительно был вызван?

- **A)** `assertThat(result).isInstanceOf(Result.Success::class.java)`
- **B)** `verify { habitRepository.saveHabit(completedHabit) }`
- **C)** `every { habitRepository.saveHabit(any()) } just Runs`
- **D)** Через `MockWebServer`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`every { ... } returns ...` задаёт поведение мока (Arrange), а `verify { habitRepository.saveHabit(completedHabit) }` проверяет факт взаимодействия (Assert). Это отдельная проверка: результат может быть успешным, но сохранение не вызвано — и тест это поймает.

**Почему другие не подходят:**
- A — проверяет тип результата, но не факт вызова метода
- C — это настройка мока, а не проверка
- D — MockWebServer используется для сетевых integration-тестов, не для моков репозитория

</details>

---

## Вопрос 7. Как Catch-сценарий «привычка не найдена» оформлен в тесте?

- **A)** `every { habitRepository.getHabit(habitId) } returns Result.Error(HabitNotFoundError)` и проверка `isInstanceOf(Result.Error::class.java)`
- **B)** `assertThat(result).isEqualTo(Result.Success)`
- **C)** `verify { dateProvider.now() }`
- **D)** Тест отсутствует — AI сгенерировал только happy path

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В тесте ``should return HabitNotFoundError when habit does not exist`` мок возвращает `Result.Error(HabitNotFoundError)`, а assertion проверяет `assertThat(result).isInstanceOf(Result.Error::class.java)`. Третий тест аналогично проверяет `Result.Error(DatabaseError)` при неудачном сохранении.

**Почему другие не подходят:**
- B — противоречит ожидаемому error-сценарию
- C — dateProvider здесь ни при чём
- D — в модуле показаны три теста, включая два error-сценария

</details>

---

## Вопрос 8. Что требует шаблон промпта для создания моков?

- **A)** Только один мок на интерфейс без настроек
- **B)** Тестовый utility-класс с factory-методами, default behavior и helper-функциями для типовых сценариев
- **C)** Реальные JDBC-подключения вместо моков
- **D)** Генерацию тестовых данных вручную в каждом тесте

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Требования шаблона: 1) test utility класс с factory methods для моков, 2) default behavior для каждого метода, 3) helper-функции для common scenarios. Практическое задание 4.2 ожидает сокращения boilerplate в тестах на 50%+.

**Почему другие не подходят:**
- A — без default behavior и helpers нет сокращения повторов
- C — реальный I/O делает тесты медленными (см. ошибку «AI создает медленные тесты»)
- D — это и есть boilerplate, от которого избавляются factory-функции

</details>

---

## Вопрос 9. Что означает `any()` в этой factory-функции?

```kotlin
fun mockHabitRepository(
    getHabitResult: Result<Habit> = Result.Success(defaultHabit),
    saveHabitResult: Result<Unit> = Result.Success
): HabitRepository = mock {
    every { getHabit(any()) } returns getHabitResult
    every { saveHabit(any()) } returns saveHabitResult
}
```

- **A)** Мок принимает только `null`
- **B)** Мок срабатывает для любого аргумента нужного типа
- **C)** Мок проверяет, что аргумент равен `defaultHabit`
- **D)** Мок отключает вызов метода

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`any()` из MockK — это matcher, который соответствует любому значению параметра. Благодаря ему фабрика задаёт default behavior независимо от конкретного `habitId`, а конкретные тесты могут переопределить результат через параметры `getHabitResult` / `saveHabitResult`.

**Почему другие не подходят:**
- A — для `null` используется `isNull()` / `anyNullable()`
- C — для конкретного значения аргумент передавали бы явно
- D — заглушка задаёт поведение, а не отключает вызов

</details>

---

## Вопрос 10. Что НЕ входит в фокус шаблона анализа coverage?

- **A)** Exception handling paths и boundary conditions
- **B)** Null/empty сценарии и concurrent race conditions
- **C)** Увеличение размера APK за счёт тестовых зависимостей
- **D)** Конкретные тест-кейсы для непокрытых строк и ожидаемый прирост coverage

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

Шаблон (роль QA Lead) просит проанализировать непокрытые строки/ветви, пропущенные edge cases и error scenarios, затем предложить конкретные кейсы, полный код новых тестов и ожидаемое улучшение coverage. Фокус: exception handling, boundary, null/empty, race conditions. Целевой coverage — 80%+. Размер APK не обсуждается.

**Почему другие не подходят:**
- A, B — прямые пункты раздела «Сосредоточься на»
- D — это раздел «Предложи»

</details>

---

## Вопрос 11. Найди ветвь (ловушка):

```kotlin
fun calculateDiscount(price: Double, userId: String?, isPremium: Boolean): Double =
    when {
        price < 0 -> throw IllegalArgumentException("Price cannot be negative")
        isPremium && userId != null -> price * 0.7
        isPremium -> price * 0.85
        userId != null && isNewUser(userId) -> price * 0.9
        else -> price
    }
```

Какой скидке соответствует `price * 0.9`?

- **A)** Премиум-пользователь с любым `userId`
- **B)** Премиум-пользователь без `userId`
- **C)** Не-премиум пользователь с `userId`, который является новым
- **D)** Любой пользователь с отрицательной ценой

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

`when` проверяет ветви по порядку. Для премиум-пользователя с `userId` первая подходящая ветвь даёт 0.7, поэтому 0.9 к нему никогда не применяется. Ветвь `price * 0.9` достигается только если `isPremium == false`, `userId != null` и `isNewUser(userId) == true`.

Всего в `when` пять ветвей, и текущие тесты покрывают только случай `isPremium = true, userId != null` (0.7) — AI должен добавить тесты для оставшихся четырёх.

**Почему другие не подходят:**
- A — это ветвь 0.7 (и она приоритетнее)
- B — это ветвь 0.85
- D — для отрицательной цены выбрасывается IllegalArgumentException

</details>

---

## Вопрос 12. Какой инструмент НЕ указан в шаблоне integration-теста?

- **A)** TestContainer для SQLite (если есть БД)
- **B)** MockWebServer или FakeHttp для сети
- **C)** StandardTestDispatcher для корутин
- **D)** Figma-to-Compose плагин

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: D**

В шаблоне integration-тестов перечислены: test databases и mock network, полный user flow, валидация состояния, TestContainer для SQLite, MockWebServer/FakeHttp и StandardTestDispatcher. Figma-плагин относится к Модулю 2 (UI) и к тестированию отношения не имеет.

**Почему другие не подходят:**
- A, B, C — все три прямо названы в шаблоне

</details>

---

## Вопрос 13. Какая пара «API — назначение» для UI-тестов верна?

- **A)** `createComposeRule()` — assertion, что элемент отображается
- **B)** `onNodeWithText()` / `onNodeWithTag()` — поиск и взаимодействие с узлами
- **C)** `assertIsDisplayed()` — настройка тестового окружения
- **D)** `assertHasClickAction()` — отправка запроса к LLM

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Шаблон UI-тестов: `createComposeRule()` используется для setup, `onNodeWithText()` / `onNodeWithTag()` — для interactions, `assertHasClickAction()` / `assertIsDisplayed()` — для assertions. Нужно покрыть initial state rendering, пользовательские действия, изменение состояния и отображение ошибок; всего 3–5 тестов.

**Почему другие не подходят:**
- A — `createComposeRule()` задаёт правило, а не проверяет отображение
- C — `assertIsDisplayed()` как раз assertion
- D — `assertHasClickAction()` проверяет кликабельность, LLM здесь ни при чём

</details>

---

## Вопрос 14. Как исправить ошибку «AI создает flaky тесты»?

- **A)** Увеличить timeout тестов в 10 раз
- **B)** Попросить: «Создай deterministic тесты без race conditions»
- **C)** Удалить все async-тесты
- **D)** Запускать тесты только вручную

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В разделе ошибок для flaky-тестов решение дано дословно: попросить детерминированные тесты без гонок. Для непокрытых edge cases — явно перечислить null, empty, negative values; для медленных тестов — использовать in-memory реализации и избегать I/O.

**Почему другие не подходят:**
- A — большой timeout маскирует проблему, а не устраняет её
- C — потеря покрытия async-логики
- D — ручной запуск не делает тест стабильным

</details>

---

## Вопрос 15. Какой инструмент предназначен для coverage именно Kotlin Multiplatform?

- **A)** JaCoCo
- **B)** Kover
- **C)** Turbine
- **D)** Accompanist Testing

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В списке инструментов: JaCoCo — code coverage для JVM, Kover — coverage для Kotlin Multiplatform. Turbine используется для тестирования Flow, Accompanist Testing — для Compose-тестов.

**Почему другие не подходят:**
- A — JaCoCo покрывает JVM, а не все цели KMP
- C — Turbine тестирует Flow, а не покрытие
- D — Accompanist помогает с Compose UI, а не с метриками coverage

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 13–15 | 🎉 Отлично! Готов автоматизировать тесты с AI и проверять их качество |
| 10–12 | 📖 Хорошо. Перечитай шаблоны промптов и раздел «Ошибки при генерации тестов» |
| 7–9 | ⚠️ Повтори темы 4.1–4.4: MockK, coverage, integration и UI-тесты |
| < 7 | 🆘 Начни с задания 4.1: сгенерируй тесты для 3 Use Cases и проверь coverage |

---

> 🔜 Дальше — [Модуль 5: AI в нативных интеграциях](../Module_05_AI_Native_Integration/content.md).

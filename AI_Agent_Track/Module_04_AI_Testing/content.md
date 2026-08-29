# Модуль 4: AI-ассистированное тестирование

## 📋 Обзор модуля

**Продолжительность:** 8 часов  
**Сложность:** Intermediate  
**Цель:** Автоматическая генерация тестов, mock объектов и анализ coverage с помощью AI

---

## 🎯 Цели обучения

После прохождения модуля вы сможете:
- ✅ Генерировать **unit-тесты** автоматически с AI
- ✅ Создавать **mock объекты** для зависимостей
- ✅ Анализировать и улучшать **test coverage** с AI
- ✅ Писать **integration и UI тесты** с AI-ассистентом

---

## 📚 Темы модуля

### Тема 4.1: Генерация unit-тестов с AI (2 часа)

#### Шаблон prompt'а для unit-тестов:

```
Ты - Senior QA Engineer, эксперт в тестировании Kotlin кода.

Задача:
Напиши comprehensive unit-тесты для следующего класса/функции.

[ВСТАВИТЬ КОД]

Требования к тестам:
1. Используй JUnit 5 + AssertJ (или Kotlin Test)
2. Тестируй happy path, edge cases и error scenarios
3. Создавай mock объекты для всех зависимостей (MockK)
4. Покрывай минимум 90% строк кода

Структура тестов:
- Arrange (подготовка данных и моков)
- Act (вызов тестируемого кода)  
- Assert (проверка результатов)

Включи тесты для:
✓ Нормальный сценарий (happy path)
✓ Edge cases (null, empty, boundary values)
✓ Error scenarios (exceptions, invalid input)
✓ Concurrent scenarios (если async код)

Назови тесты описательно: should_[expectedBehavior]_when_[condition]
```

---

#### Пример: Тесты для Use Case

**Промпт:**
```
Напиши unit-тесты для CompleteHabitUseCase.

Код Use Case:
```kotlin
class CompleteHabitUseCase(
    private val habitRepository: HabitRepository,
    private val dateProvider: DateProvider
) {
    suspend operator fun invoke(habitId: String): Result<Habit> {
        // Валидация, логика завершения привычки, обновление streak
    }
}
```

Требования:
- MockK для habitRepository и dateProvider
- Тестируй: успешное завершение, невалидный ID, ошибка repository
- 90%+ coverage

Используй JUnit 5 + AssertJ.
```

**Ожидаемый результат от AI:**

```kotlin
class CompleteHabitUseCaseTest {
    
    private val habitRepository = mock<HabitRepository>()
    private val dateProvider = mock<DateProvider>()
    
    private lateinit var useCase: CompleteHabitUseCase
    
    @BeforeTest
    fun setup() {
        useCase = CompleteHabitUseCase(habitRepository, dateProvider)
    }
    
    @Test
    fun `should complete habit and increment streak when habit exists`() = runTest {
        // Arrange
        val habitId = "habit-123"
        val existingHabit = Habit(
            id = habitId,
            title = "Test Habit",
            streak = 5,
            isCompleted = false
        )
        
        val completedHabit = existingHabit.copy(
            isCompleted = true,
            streak = 6
        )
        
        every { habitRepository.getHabit(habitId) } returns Result.Success(existingHabit)
        every { habitRepository.saveHabit(completedHabit) } returns Result.Success
        
        // Act
        val result = useCase(habitId)
        
        // Assert
        result.fold(
            onSuccess = { habit ->
                assertThat(habit.id).isEqualTo(habitId)
                assertThat(habit.isCompleted).isTrue()
                assertThat(habit.streak).isEqualTo(6)
            },
            onFailure = { error ->
                fail("Expected success but got error: $error")
            }
        )
        
        verify { habitRepository.saveHabit(completedHabit) }
    }
    
    @Test
    fun `should return HabitNotFoundError when habit does not exist`() = runTest {
        // Arrange
        val habitId = "non-existent"
        
        every { habitRepository.getHabit(habitId) } returns Result.Error(HabitNotFoundError)
        
        // Act
        val result = useCase(habitId)
        
        // Assert
        assertThat(result).isInstanceOf(Result.Error::class.java)
    }
    
    @Test  
    fun `should return RepositoryError when save fails`() = runTest {
        // Arrange
        val habitId = "habit-123"
        val existingHabit = Habit(id = habitId, title = "Test", streak = 5, isCompleted = false)
        val completedHabit = existingHabit.copy(isCompleted = true, streak = 6)
        
        every { habitRepository.getHabit(habitId) } returns Result.Success(existingHabit)
        every { habitRepository.saveHabit(completedHabit) } returns Result.Error(DatabaseError)
        
        // Act
        val result = useCase(habitId)
        
        // Assert
        assertThat(result).isInstanceOf(Result.Error::class.java)
    }
}
```

---

### Тема 4.2: Создание mock объектов с AI (1.5 часа)

#### Шаблон prompt'а для моков:

```
Ты - Senior разработчик, эксперт в тестировании.

Задача:
Создай mock реализации для следующих интерфейсов используя MockK.

[ВСТАВИТЬ ИНТЕРФЕЙСЫ]

Требования:
1. Создай test utility класс с factory methods для моков
2. Настрой default behavior для каждого метода
3. Создай helper функции для common scenarios

Пример:
```kotlin
fun mockHabitRepository(
    getHabitResult: Result<Habit> = Result.Success(defaultHabit),
    saveHabitResult: Result<Unit> = Result.Success
): HabitRepository {
    return mock<HabitRepository> {
        every { getHabit(any()) } returns getHabitResult
        every { saveHabit(any()) } returns saveHabitResult
    }
}
```

Создай аналогичные helpers для всех интерфейсов.
```

---

### Тема 4.3: Анализ и улучшение test coverage с AI (2 часа)

#### Шаблон prompt'а для анализа coverage:

```
Ты - QA Lead, эксперт в тестировании.

Контекст:
- Текущий test coverage: [X]%
- Целевой coverage: 80%+

Задача:
Проанализируй следующий код и предложи тесты для непокрытых строк.

[ВСТАВИТЬ КОД]
[ВСТАВИТЬ ТЕКУЩИЕ ТЕСТЫ]

Анализируй:
1. Какие строки/ветви не покрыты тестами?
2. Какие edge cases пропущены?
3. Какие error scenarios не протестированы?

Предложи:
1. Конкретные тест-кейсы для непокрытого кода
2. Полный код новых тестов
3. Ожидаемое улучшение coverage после добавления

Сосредоточься на:
- Exception handling paths
- Boundary conditions  
- Null/empty scenarios
- Concurrent race conditions (если async)
```

---

#### Пример анализа coverage:

**Промпт:**
```
Анализируй этот код и найди непокрытые ветви:

```kotlin
fun calculateDiscount(price: Double, userId: String?, isPremium: Boolean): Double {
    return when {
        price < 0 -> throw IllegalArgumentException("Price cannot be negative")
        isPremium && userId != null -> price * 0.7
        isPremium -> price * 0.85  
        userId != null && isNewUser(userId) -> price * 0.9
        else -> price
    }
}
```

Текущие тесты покрывают только isPremium = true, userId != null case.
Напиши тесты для всех остальных ветвей.
```

**Ожидаемый результат:** AI создаст тесты для всех 5 ветвей when expression.

---

### Тема 4.4: Integration и UI тесты с AI (2 часа)

#### Шаблон prompt'а для integration тестов:

```
Ты - Senior QA Engineer.

Задача:
Создай integration тест для [ФУНКЦИОНАЛ] который тестирует полный flow.

Контекст:
- Тестируем взаимодействие между [КОМПОНЕНТ 1], [КОМПОНЕНТ 2]
- Используем TestDispatcher для корутин

Требования:
1. Настрой тестовое окружение (test databases, mock network)
2. Протестируй полный user flow от начала до конца
3. Валидируй состояние после выполнения

Используй:
- TestContainer для SQLite (если DB)
- MockWebServer или FakeHttp для network
- StandardTestDispatcher для корутин

Напиши полный тест с setup, execution и validation.
```

---

#### Шаблон prompt'а для UI тестов:

```
Ты - Senior Android/iOS Test Engineer.

Задача:
Создай UI тест для [SCREEN/COMPONENT] используя Compose Testing.

Требования:
1. Создай тестовый composable с test doubles
2. Напиши тесты для:
   - Initial state rendering
   - User interactions (clicks, text input)
   - State changes и UI updates
   - Error states отображение

Используй:
- createComposeRule() для setup
- onNodeWithText(), onNodeWithTag() для interactions
- assertHasClickAction(), assertIsDisplayed() для assertions

Напиши 3-5 comprehensive UI тестов.
```

---

## 📝 Практические задания модуля

### Задание 4.1: Генерация тестов для Use Cases (2 часа)

**Требования:**
- Создайте unit-тесты для 3-5 Use Cases из EcoTrack
- Достижение 90%+ coverage

**Критерии:**
- ✓ Все happy path и error scenarios покрыты
- ✓ Тесты быстрые (<100ms каждый)

---

### Задание 4.2: Создание mock utilities (1.5 часа)

**Требования:**
- Создайте factory functions для моков всех репозиториев
- Добавьте helpers для common test scenarios

**Критерии:**
- ✓ Уменьшение boilerplate в тестах на 50%+

---

### Задание 4.3: Улучшение test coverage (2 часа)

**Требования:**
- Проанализируйте текущий coverage с AI
- Добавьте тесты для непокрытого кода

**Критерии:**
- ✓ Увеличение coverage с X% до 80%+

---

### Задание 4.4: Integration и UI тесты (2 часа)

**ТребRequirements:**
- 1 integration test для полного flow
- 3 UI теста для основных screens

**Критерии:**
- ✓ Тесты стабильные и воспроизводимые

---

## 🚫 Ошибки при генерации тестов с AI

### ❌ AI создает flaky тесты
**Решение:** Просите: "Создай deterministic тесты без race conditions"

### ❌ AI не покрывает edge cases
**Решение:** Явно перечисляйте: "Включи тесты для null, empty, negative values"

### ❌ AI создает медленные тесты
**Решение:** Указывайте: "Используй in-memory implementations, избегай I/O"

---

## 📚 Дополнительные материалы

### Фреймворки:
- [MockK](https://mockk.io/) - Mocking для Kotlin Multiplatform
- [Turbine](https://github.com/cashapp/turbine) - Testing Flow
- [Accompanist Testing](https://google.github.io/accompanist/testing/) - Compose testing

### Книги:
- "Kotlin Testing with JUnit 5" (2024)
- "Effective Unit Testing in Kotlin"

### Инструменты:
- [JaCoCo](https://www.jacoco.org/) - Code coverage для JVM
- [Kover](https://kotlin.github.io/kover/) - Coverage для Kotlin Multiplatform

---

## 🚀 Следующий шаг

Переходите к [Модулю 5](../Module_05_AI_Native_Integration/content.md): AI в нативных интеграциях

**Время до следующего модуля:** 1-2 недели  
**Рекомендуемая практика:** Пишите тесты для каждого нового фича с AI

---

**Удачи в автоматизации тестирования с помощью ИИ! 🧪🤖**

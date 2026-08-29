# Prompt Library для KMP разработки

## 📚 Библиотека готовых prompt'ов

Эта библиотека содержит проверенные и эффективные prompt'ы для различных задач в KMP разработке с AI.

---

## 🏗️ Архитектура и структура проекта

### Prompt 1: Создание нового KMP проекта

```
Ты - Senior Kotlin Multiplatform разработчик.

Создай структуру нового KMP проекта для [ТИП ПРИЛОЖЕНИЯ - например: e-commerce, social media, fitness tracker].

Требования:
- Feature-first modularization (каждая фича - отдельный модуль)
- Clean Architecture (Domain, Data, UI слои в каждом feature module)
- Shared core modules для common functionality

Структура:
app/ (Android entry point)
iosApp/ (iOS entry point)  
shared/
  core/
    - common (Result, Either, extensions, coroutines config)
    - network (Ktor client setup, interceptors, API clients)
    - db (SQLDelight setup, migrations, query providers)  
    - designsystem (Material 3 theme, reusable components)
    - di (Koin modules)
  features/
    feature-auth/ (domain, data, ui)
    feature-home/ (domain, data, ui)
    [другие features...]

Для каждого модуля создай:
1. build.gradle.kts с правильными dependencies
2. Package structure
3. Main entry point или module setup class

Используй:
- Kotlin 1.9+
- Compose Multiplatform 1.5+
- Ktor 2.3+
- SQLDelight 2.0+
- Koin 3.5+

Покажи полную структуру проекта с файлами и кратким описанием каждого модуля.
```

---

### Prompt 2: Проектирование Clean Architecture для фичи

```
Ты - Principal Architect с опытом в Clean Architecture.

Спроектируй Clean Architecture для фичи "[НАЗВАНИЕ ФИЧИ - например: User Profile]".

Контекст:
- KMP приложение с feature-first modularization  
- Ожидаемая сложность: [низкая/средняя/высокая]
- Критические требования: [offline support, real-time updates, etc.]

Создай архитектуру с:
1. **Domain Layer** (business logic, без зависимостей от frameworks)
   - Use Cases (интерфейсы и реализации)
   - Entities (чистые domain objects)
   - Repository interfaces
   - Domain errors sealed class

2. **Data Layer** (implementation details)  
   - Repository implementations
   - Local data source (SQLDelight)
   - Remote data source (Ktor API client)
   - DTOs и mappers (Domain <-> Data)

3. **UI Layer** (presentation)
   - ViewModel с StateFlow
   - Composable screens и components  
   - UI state data classes

Для каждого слоя покажи:
- Package structure
- Key classes и их ответственность
- Dependencies (что от чего зависит)

Обоснуй архитектурные решения и покажи Mermaid diagram для визуализации.
```

---

## 💾 Data Layer и Persistence

### Prompt 3: Создание Repository с SQLDelight

```
Ты - Senior KMP разработчик, эксперт в SQLDelight и data layer patterns.

Создай repository для [ENTITY NAME - например: User, Product, Order].

Entity поля:
- id: String (UUID)
- [field1]: [type] - [description]  
- [field2]: [type] - [description]
- createdAt: Instant
- updatedAt: Instant

Требования к repository:
1. CRUD operations (create, read, update, delete)
2. Return Result<T, Error> для всех operations  
3. Support pagination с cursor-based approach
4. In-memory caching с TTL (5 minutes)
5. Flow для reactive updates (observeById, observeAll)

Создай:
1. SQLDelight schema (.sq file) с таблицей и indexes
2. QueryProvider interface с CRUD queries  
3. Repository implementation с caching strategy
4. Mappers между DbEntity, Dto и Domain Entity

Используй:
- SQLDelight 2.0+ с coroutines support
- Proper error handling (DatabaseError sealed class)
- Transaction для multi-step operations

Покажи полный код всех компонентов.
```

---

### Prompt 4: Создание API Client с Ktor

```
Ты - Senior Backend-to-Mobile разработчик, эксперт в Ktor client.

Создай API client для [API NAME - например: User API, Product API].

Требования:
1. REST API client с Ktor  
2. Support для: GET, POST, PUT, DELETE endpoints
3. Automatic error mapping (HTTP errors -> Domain errors)
4. Request/response serialization с kotlinx.serialization
5. Interceptors для: auth tokens, logging, retry policy

Endpoints:
- GET /api/[resource] - list с pagination
- GET /api/[resource]/{id} - get by ID  
- POST /api/[resource] - create
- PUT /api/[resource]/{id} - update
- DELETE /api/[resource]/{id} - delete

Создай:
1. API interface с suspend functions
2. DTOs для request/response (@Serializable)  
3. Ktor client configuration с interceptors
4. Error mapping (HttpError -> DomainError)

Используй:
- Ktor client 2.3+ с ContentNegotiation
- kotlinx.serialization для JSON
- Result sealed class для error handling

Покажи полный код API client.
```

---

## 🎨 UI и Compose Multiplatform

### Prompt 5: Создание Composable компонента

```
Ты - Senior UI разработчик, эксперт в Compose Multiplatform.

Создай reusable @Composable компонент "[КОМПОНЕНТ NAME - например: ProductCard, UserAvatar]".

Контекст:
- Material 3 design system
- Support для light/dark themes  
- Multiplatform (Android, iOS, Desktop)

Требования к компоненту:
1. State hoisting (все state в parent, component receives via parameters)
2. Event callbacks (onClick, onLongClick, etc.)
3. Accessibility support (contentDescription, focus management)
4. Responsive design (adapt to different screen sizes)

Component должен отображать:
- [field1]: [description]
- [field2]: [description]  
- [actions]: [button, menu, etc.]

Создай:
1. Data class для component state (если нужно)
2. @Composable функцию с параметрами и callbacks
3. Пример использования в parent screen
4. Preview annotation для IDE preview

Используй Material 3 компоненты (Card, Button, Text, Icon).
Покажи полный код с KDoc комментариями.
```

---

### Prompt 6: Создание Screen с ViewModel

```
Ты - Senior KMP разработчик, эксперт в Compose UI и MVVM.

Создай screen "[SCREEN NAME - например: ProductDetailScreen]" с ViewModel.

Требования к UI:
1. Composable screen function
2. Accept viewModel как parameter (dependency injection)  
3. Collect StateFlow и отображать UI state
4. Handle loading, success, error states
5. Respond to user events

Требования к ViewModel:
1. ViewModel class с StateFlow<State>
2. Initial state (Loading)
3. Load data on initialization или по event  
4. Handle errors gracefully
5. Expose events для navigation и user feedback

UI State sealed class:
- Loading
- Success(data: [DataType])  
- Error(message: String)

User Events:
- [Event1]: [description]
- [Event2]: [description]

Создай полный код screen + ViewModel с proper state management.
```

---

## 🧪 Testing

### Prompt 7: Генерация Unit Tests для Use Case

```
Ты - Senior QA Engineer, эксперт в тестировании Kotlin кода.

Напиши comprehensive unit-тесты для Use Case: [USE CASE NAME]

Use Case код:
[ВСТАВИТЬ КОД USE CASE]

Требования к тестам:
1. JUnit 5 + AssertJ (или Kotlin Test)
2. MockK для всех зависимостей  
3. Тестируй: happy path, edge cases, error scenarios
4. 90%+ code coverage

Структура каждого теста:
- Arrange (setup mocks и test data)
- Act (invoke use case)  
- Assert (verify results и mock interactions)

Назови тесты описательно:
should_[expectedBehavior]_when_[condition]

Включи тесты для:
✓ Нормальный сценарий (все зависимости работают корректно)
✓ Edge cases (null, empty, boundary values)  
✓ Error scenarios (repository errors, validation failures)
✓ Concurrent scenarios (если async operations)

Покажи полный код тестового класса.
```

---

### Prompt 8: Создание Mock Factory Functions

```
Ты - Senior разработчик, эксперт в тестировании.

Создай test utility класс с factory functions для mock объектов.

Интерфейсы для моков:
[ВСТАВИТЬ ИНТЕРФЕЙСЫ - например: UserRepository, ApiService]

Требования:
1. Factory function для каждого interface
2. Default behavior для всех методов (reasonable defaults)
3. Override parameters для customization
4. Helper functions для common scenarios

Пример:
```kotlin
fun mockUserRepository(
    getUserResult: Result<User> = Result.Success(defaultUser),
    saveUserResult: Result<Unit> = Result.Success,
    usersFlow: Flow<List<User>> = flow { emit(emptyList()) }
): UserRepository {
    return mock<UserRepository> {
        every { getUser(any()) } returns getUserResult
        every { saveUser(any()) } returns saveUserResult  
        every { observeUsers() } returns usersFlow
    }
}

fun mockUserRepositoryWithError(error: DomainError): UserRepository {
    return mock<UserRepository> {
        every { getUser(any()) } returns Result.Error(error)
        every { saveUser(any()) } returns Result.Error(error)
    }
}
```

Создай аналогичные factory functions для всех интерфейсов.
Включи helpers для common test scenarios (success, error, empty).
```

---

## 🔐 Security и Production

### Prompt 9: Security Audit для KMP кода

```
Ты - Mobile Security Engineer с 10+ лет опытом.

Проведи comprehensive security audit для следующего KMP кода:

[ВСТАВИТЬ КОД]

Анализируй следующие аспекты:

## 1. Data Security
- Encryption at rest (database, preferences)  
- Encryption in transit (TLS, certificate pinning)
- Sensitive data storage (API keys, tokens, PII)

## 2. Authentication & Authorization
- Token storage и rotation  
- Session management
- Permission escalation vulnerabilities

## 3. Input Validation
- SQL injection prevention  
- XSS prevention (если есть HTML rendering)
- Command injection

## 4. Error Handling  
- No sensitive data in error messages
- Proper logging без leakage

## 5. Platform-Specific Security
- Android: Backup prevention, flagSecure
- iOS: Keychain usage, sandboxing

Для каждого найденного issue покажи:
- Severity (Critical/High/Medium/Low)
- Description и почему это уязвимость  
- Exploit scenario (как могут использовать)
- Fix с конкретным кодом

Создай prioritized security audit report.
```

---

### Prompt 10: Создание Secure Storage Solution

```
Ты - Security Engineer, эксперт в secure data storage.

Создай secure storage solution для KMP приложения.

Требования:
1. Store sensitive data (API tokens, user credentials) securely
2. Android: EncryptedSharedPreferences или Jetpack Security  
3. iOS: Keychain через Kotlin/Native bindings
4. Auto-encryption с strong algorithms (AES-256)
5. Biometric unlock option

Создай expect/actual implementation:

Expect interface (commonMain):
```kotlin
interface SecureStorage {
    suspend fun save(key: String, value: String): Result<Unit>
    suspend fun load(key: String): Result<String?>  
    suspend fun delete(key: String): Result<Unit>
    suspend fun saveWithBiometric(key: String, value: String): Result<Unit>
}
```

Actual implementations:
- Android: Jetpack Security EncryptedSharedPreferences
- iOS: Keychain с proper accessibility settings

Включи error handling и fallback mechanisms.
Покажи полный код для всех платформ.
```

---

## 🤖 Machine Learning Integration

### Prompt 11: ML Kit / Core ML Integration

```
Ты - Mobile ML Engineer с опытом в on-device machine learning.

Интегрируй [ML ФУНКЦИОНАЛ - например: Image Classification, Text Recognition] в KMP приложение.

Контекст:
- Android: ML Kit от Google  
- iOS: Core ML + Create ML

Требования к expect interface (commonMain):
1. Model initialization и loading
2. Inference method: predict(input): Result<Output>  
3. Model metadata (version, supported inputs)
4. Error handling для missing models, invalid input

[ОПИСАНИЕ ML ФУНКЦИОНАЛА]
- Input type: [например: Image, Text, Audio]
- Output type: [например: Classification labels, Bounding boxes, Transcribed text]
- Supported languages/formats: [если применимо]

Создай:
1. Expect interface в commonMain с input/output data classes
2. Android actual implementation с ML Kit API  
3. iOS actual implementation с Core ML (.mlmodelc)
4. Instructions для добавления model файлов в проекты
5. Example usage code

Покажи полный код и configuration files (build.gradle, Info.plist).
```

---

## 📝 Как использовать эту библиотеку

1. **Выберите подходящий prompt** для вашей задачи
2. **Заполните placeholders** в квадратных скобках [...]
3. **Скопируйте и запустите** prompt в вашем AI tool
4. **Review сгенерированный код** перед использованием
5. **Адаптируйте под ваш контекст** проекта

## 🔄 Вклад в библиотеку

Отправьте PR с новыми prompt'ами:
1. Создайте новый файл в этой директории или добавьте в существующий
2. Тестируйте prompt перед submission  
3. Включите пример использования и ожидаемый результат
4. Укажите категорию и сложность

## 📖 Дополнительные ресурсы

- [Module 1](../Module_01_AI_Setup/content.md) - Настройка AI инструментов
- [Module 2](../Module_02_AI_Code_Generation/content.md) - Генерация кода
- [Advanced Prompt Engineering](https://www.promptengineering.org/)

---

**Happy AI-assisted coding! 🤖💻**

# Prompt'ы для генерации KMP кода

## 📦 Data Classes и Models

### Prompt: Создание data class с validation

```
Ты - Senior Kotlin разработчик.

Создай data class "[CLASS NAME]" для представления [ОПИСАНИЕ - например: User, Product].

Поля:
- id: String (UUID) - [description]
- [field1]: [type] - [description, constraints]
- [field2]: [type] - [description, constraints]  
- createdAt: Instant
- updatedAt: Instant

Требования:
1. @Serializable annotation (kotlinx.serialization)
2. Validation через constructor или companion object function
3. Custom validation errors sealed class  
4. Copy with validation (validatedCopy function)
5. ToString с masking sensitive fields

Validation rules:
- [rule1]: [description]
- [rule2]: [description]

Используй Result<T, ValidationError> для validation.
Покажи полный код с KDoc комментариями.
```

---

### Prompt: Создание DTO и Domain Entity mappers

```
Ты - Senior разработчик, эксперт в Clean Architecture.

Создай mappers между:
- API DTO (от network layer)  
- DB Entity (для SQLDelight)
- Domain Entity (business logic)

Entity: [ENTITY NAME]

DTO поля (от API):
- id: String
- [field1]: [type]  
- [field2]: [type]

DB Entity поля (для SQLDelight):
- id: String  
- [field1]: [type]
- [field2]: [type]
- created_at: Long (timestamp)

Domain Entity поля (business logic):
- id: String
- [field1]: [type]  
- [field2]: [type]
- createdAt: Instant

Создай:
1. Mapper interface с extension functions
2. DTO -> Domain Entity mapper  
3. DB Entity -> Domain Entity mapper
4. Domain Entity -> DB Entity mapper (для save)

Используй:
- Extension functions на классах
- Validation при mapping  
- Proper error handling

Покажи полный код всех mappers.
```

---

## 🏗️ Use Cases и Business Logic

### Prompt: Создание Use Case с validation

```
Ты - Senior разработчик, эксперт в Clean Architecture.

Создай Use Case "[USE CASE NAME]" для [ОПИСАНИЕ БИЗНЕС-ЛОГИКИ].

Входные параметры:
- [param1]: [type] - [description, validation rules]
- [param2]: [type] - [description, validation rules]

Зависимости:
- [repository1]: [interface] - [что делает]
- [repository2]: [interface] - [что делает]

Бизнес-логика:
1. Validate input parameters  
2. [business logic step 1]
3. [business logic step 2]
4. Save results к repository
5. Return success или error

Return type: Result<[Output Type], [Error Type]>

Создай:
1. Use Case interface (functional interface с invoke)
2. Implementation class с dependency injection  
3. Input data class с validation
4. Output data class (если нужно)
5. Error sealed class для этого use case

Следуй Single Responsibility Principle.
Покажи полный код.
```

---

### Prompt: Создание Use Case с caching strategy

```
Ты - Senior разработчик, эксперт в offline-first architecture.

Создай Use Case "[USE CASE NAME]" с intelligent caching.

Контекст:
- Data fetched от remote API  
- Cached locally для offline access
- Cache TTL: [TIME - например: 5 minutes]

Требования к caching strategy:
1. Check cache first (если не expired)
2. If cache miss или expired, fetch from network  
3. Update cache с new data
4. Return cached data immediately (stale-while-revalidate)
5. Background refresh cache

Создай:
1. Use Case с caching logic  
2. Cache manager для this data type
3. Cache metadata (timestamp, version)
4. Network fallback mechanism

Используй:
- In-memory cache (ConcurrentHashMap или similar)  
- TTL-based expiration
- Result.Success/Error для error handling

Покажи полный код с объяснением caching strategy.
```

---

## 📡 Network Layer

### Prompt: Создание API client с retry policy

```
Ты - Senior Backend-to-Mobile разработчик.

Создай API client для [API NAME] с advanced retry policy.

Endpoints:
- GET /api/[resource] - [description]
- POST /api/[resource] - [description]

Retry policy требования:
1. Exponential backoff (1s, 2s, 4s, 8s)
2. Max retries: 3
3. Retry только на transient errors (503, timeout, network error)  
4. Don't retry на 4xx errors (client errors)
5. Callback on each retry attempt

Error handling:
- Map HTTP errors к domain errors  
- Network errors (no connection, timeout)
- Serialization errors

Создай:
1. Ktor client configuration с retry interceptor  
2. API interface с suspend functions
3. Error mapping logic
4. Retry policy configuration

Используй Ktor client 2.3+ с proper interceptors.
Покажи полный код.
```

---

### Prompt: Создание API client с authentication

```
Ты - Senior разработчик, эксперт в secure API integration.

Создай authenticated API client для [API NAME].

Authentication requirements:
1. Bearer token authentication  
2. Auto token refresh на 401 Unauthorized
3. Token storage через SecureStorage
4. Token validation перед requests

Endpoints:
- GET /api/[resource] - requires auth
- POST /api/[resource] - requires auth

Token refresh flow:
1. On 401, check if token can be refreshed  
2. Call /api/auth/refresh с refresh token
3. Update stored access token
4. Retry original request once

Создай:
1. Auth interceptor для Ktor client  
2. Token refresh logic
3. Secure token storage integration
4. API interface с authenticated endpoints

Включи proper error handling для auth failures.
Покажи полный код.
```

---

## 🎨 UI Components и Screens

### Prompt: Создание reusable Card component

```
Ты - Senior UI разработчик, эксперт в Compose Multiplatform.

Создай reusable Card component "[CARD NAME]Card" для отображения [ENTITY TYPE].

Component должен отображать:
- Primary content: [field1]  
- Secondary content: [field2, field3]
- Image/Avatar (optional)
- Action button(s): [button1, button2]

Requirements:
1. State hoisting (все state в parent)  
2. Event callbacks (onClick, onLongClick, onActionClick)
3. Loading state support (skeleton loader)
4. Error state support  
5. Empty state support

Design requirements:
- Material 3 Card component
- Support light/dark themes  
- Responsive (adapt to container width)
- Accessibility (contentDescription, focus)

Создай:
1. Data class для card state  
2. @Composable function с параметрами и callbacks
3. Preview annotations для IDE preview
4. Example usage в parent screen

Покажи полный код с KDoc комментариями.
```

---

### Prompt: Создание List Screen с pagination

```
Ты - Senior KMP разработчик, эксперт в Compose UI.

Создай List Screen "[SCREEN NAME]Screen" с infinite scrolling и pagination.

Requirements:
1. LazyColumn для list rendering  
2. Cursor-based pagination (load more on scroll)
3. Pull-to-refresh support
4. Loading states (initial, loading more, refreshing)
5. Empty state когда no data
6. Error state с retry button

ViewModel requirements:
1. StateFlow<State> с sealed class states  
2. Load initial data on creation
3. Load more when user scrolls to end
4. Refresh на pull-to-refresh gesture
5. Proper error handling и retry logic

UI State sealed class:
- Initial  
- Loading(isRefreshing: Boolean)
- Success(items: List<[Item]>, hasMore: Boolean, cursor: String?)
- Error(message: String)

Создай полный код Screen + ViewModel с pagination logic.
Используй LazyListState для detection of end of list.
```

---

## 🔧 Utility и Helpers

### Prompt: Создание Result extension functions

```
Ты - Senior Kotlin разработчик, эксперт в functional programming.

Создай comprehensive set of extension functions для Result<T, E> sealed class.

Result definition:
```kotlin
sealed class Result<out T, out E> {
    data class Success<out T>(val data: T) : Result<T, Nothing>()
    data class Error<out E>(val exception: E) : Result<Nothing, E>()
}
```

Создай extension functions для:
1. map { Success -> transform data, Error -> pass through }  
2. mapError { Success -> pass through, Error -> transform error }
3. flatMap { Success -> suspend function returning Result, Error -> pass through }
4. getOrNull() - return data or null
5. fold(onSuccess: (T) -> Unit, onError: (E) -> Unit)
6. onSuccess { (T) -> Unit } - execute only on success  
7. onError { (E) -> Unit } - execute only on error
8. chain { suspend () -> Result<Unit, E> } - execute additional operation

Используй Kotlin coroutines для suspend versions.
Покажи полный код с KDoc и примерами использования.
```

---

### Prompt: Создание Date/Time helpers

```
Ты - Senior разработчик, эксперт в date/time handling.

Создай utility class с helpers для работы с датами и временем в KMP.

Requirements:
1. Format Instant to human-readable string (relative time: "2 minutes ago")  
2. Parse string to Instant (support multiple formats)
3. Calculate time difference between Instants  
4. Check if date is within range
5. Generate ISO 8601 formatted strings

Multiplatform considerations:
- Use kotlinx-datetime (Instant, LocalDateTime)  
- Avoid platform-specific date APIs in commonMain
- Handle timezones properly

Создай:
1. DateTimeFormatter utility class  
2. Extension functions на Instant
3. Date range validation helpers

Покажи полный код с примерами использования.
```

---

## 📝 Как использовать эти prompt'ы

1. **Выберите категорию** (Data Classes, Use Cases, Network, UI)
2. **Выберите конкретный prompt** для вашей задачи  
3. **Заполните placeholders** в [...] вашими данными
4. **Скопируйте и запустите** prompt в AI tool
5. **Review и адаптируйте** сгенерированный код

## 🔄 Внесите свой вклад

Отправьте PR с новыми prompt'ами:
- Тестируйте перед submission  
- Включите примеры использования
- Укажите expected output

---

**Happy coding! 🤖💻**

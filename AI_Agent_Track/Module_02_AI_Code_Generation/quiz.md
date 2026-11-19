# 📝 Самопроверка: Модуль 2 — Генерация KMP кода с AI

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 12+ из 15 без подглядывания.

---

## Вопрос 1. Что **НЕ** входит в цели обучения модуля?

- **A)** Генерировать expect/actual реализации для платформенных API
- **B)** Создавать Compose UI компоненты с AI
- **C)** Автоматически создавать модели данных и DTO
- **D)** Публиковать готовое приложение в App Store и Google Play

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: D**

Цели модуля (8 часов, Beginner-Intermediate): генерировать expect/actual для платформенных API, создавать Compose UI компоненты, автоматически создавать модели данных и DTO, писать базовую бизнес-логику с AI-ассистентом. Публикация в сторы в цели не входит.

**Почему не другие:**
- A — это тема 2.1 (2 часа)
- B — тема 2.2 (2 часа)
- C — тема 2.3 (1.5 часа); ещё есть тема 2.4 — бизнес-логика

</details>

---

## Вопрос 2. В шаблоне промпта для expect/actual что допускается для Desktop-реализации?

```text
Требования к actual:
- Android: используй [ANDROID API]
- iOS: используй [iOS FRAMEWORK]
- Desktop: fallback implementation или throw NotImplementedError
```

- **A)** Вообще не создавать `actual` для Desktop
- **B)** Fallback implementation или `throw NotImplementedError`
- **C)** Скопировать Android-реализацию без изменений
- **D)** Вернуть `null` из всех suspend-методов

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Шаблон прямо допускает для Desktop упрощённый вариант: fallback implementation или `throw NotImplementedError`. Это честный способ обозначить неподдерживаемую платформу вместо имитации работы.

**Почему не другие:**
- A — без `actual` common-объявление не скомпилируется для Desktop
- C — Android API (`Context`, `NotificationManager`) на Desktop недоступны
- D — возврат `null` маскирует проблему и не соответствует шаблону

</details>

---

## Вопрос 3. Что вернёт `sendNotification` в сгенерированном коде?

```kotlin
// commonMain/platform/NotificationService.kt
sealed class NotificationResult {
    data class Success(val notificationId: String) : NotificationResult()
    data class Error(val message: String, val errorCode: Int? = null) : NotificationResult()
}

interface NotificationService {
    suspend fun sendNotification(
        title: String,
        body: String,
        data: Map<String, String> = emptyMap()
    ): NotificationResult
    suspend fun requestPermission(): PermissionStatus
    fun getPermissionStatus(): Flow<PermissionStatus>
    fun cancelNotification(id: Int)
}

expect fun NotificationService(): NotificationService
```

- **A)** `Boolean` — отправлено/не отправлено
- **B)** `String` с идентификатором уведомления
- **C)** `NotificationResult` — `Success(notificationId)` или `Error(message, errorCode)`
- **D)** `Flow<PermissionStatus>`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

Шаблон требует «обработку ошибок через sealed class Result», и в примере это `NotificationResult` с двумя вариантами: `Success(val notificationId: String)` и `Error(val message: String, val errorCode: Int? = null)`. Так вызывающий получает типизированный результат вместо исключений.

**Почему не другие:**
- A — булево значение не несёт ни id, ни причину ошибки
- B — id есть только внутри `Success`
- D — `Flow<PermissionStatus>` возвращает `getPermissionStatus()`, а не отправку

</details>

---

## Вопрос 4. Ловушка: AI показал «полный готовый код» для Android:

```kotlin
actual suspend fun requestPermission(): PermissionStatus {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        // Android 13+ требует отдельное разрешение
        TODO("Implement POST_NOTIFICATIONS permission")
    } else {
        PermissionStatus.GRANTED
    }
}

actual fun getPermissionStatus(): Flow<PermissionStatus> {
    return flow {
        emit(PermissionStatus.GRANTED) // Упрощенная версия
    }.cacheIn(/* coroutineScope */)
}
```

Можно ли считать это готовым к использованию кодом?

- **A)** Да, AI сгенерировал полный код — можно собирать релиз
- **B)** Нет: `TODO(...)` выбросит исключение, а заглушка `GRANTED` и пустой `cacheIn` требуют ручной доработки
- **C)** Да, но только на Android 13+
- **D)** Нет, потому что `sealed class` нельзя использовать в actual

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Это вопрос-ловушка. В самом примере из модуля видно: `TODO("Implement POST_NOTIFICATIONS permission")` — это не реализация, а явная заглушка, которая при вызове упадёт. `flow { emit(GRANTED) }` помечен как «Упрощенная версия», а `cacheIn(/* coroutineScope */)` — тоже незаполненное место. Сгенерированный код всегда нужно ревьюить и дорабатывать.

**Почему не другие:**
- A — опасное доверие AI на 100%
- C — именно на Android 13+ заглушка и сработает
- D — `sealed class` здесь ни при чём; проблема в незавершённых местах

</details>

---

## Вопрос 5. Какую проверку версии Android использует сгенерированный код при создании канала уведомлений?

```kotlin
private fun createNotificationChannel() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        val channel = NotificationChannel(
            "ecotrack_channel",
            "EcoTrack Notifications",
            NotificationManager.IMPORTANCE_HIGH
        )
        notificationManager.createNotificationChannel(channel)
    }
}
```

- **A)** `Build.VERSION_CODES.O` (Android 8, API 26)
- **B)** `Build.VERSION_CODES.TIRAMISU` (Android 13)
- **C)** `Build.VERSION_CODES.LOLLIPOP` (Android 5)
- **D)** Проверки нет — канал создаётся всегда

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`NotificationChannel` появился в Android 8 (API 26, `Build.VERSION_CODES.O`), поэтому пример и содержит проверку `SDK_INT >= O`. Канал создаётся с `IMPORTANCE_HIGH` и id `ecotrack_channel`; тот же id используется в `NotificationCompat.Builder`.

**Почему не другие:**
- B — `TIRAMISU` проверяется отдельно для разрешения POST_NOTIFICATIONS
- C — слишком ранняя версия
- D — без проверки код упадёт на старых Android

</details>

---

## Вопрос 6. Какое разрешение упоминается для Android 13+ в примере с уведомлениями?

- **A)** `CAMERA`
- **B)** `READ_EXTERNAL_STORAGE`
- **C)** `ACCESS_FINE_LOCATION`
- **D)** `POST_NOTIFICATIONS`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: D**

Комментарий в коде: «Android 13+ требует отдельное разрешение», а `TODO` называется `Implement POST_NOTIFICATIONS permission`. Проверка — `Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU`.

**Почему не другие:**
- A — камера здесь не нужна
- B — чтение хранилища в модуле не обсуждается
- C — геолокация относится к заданию 2.1 (Location Service)

</details>

---

## Вопрос 7. Как iOS-реализация из модуля обрабатывает статусы `Provisional` и `Ephemeral`?

```kotlin
center.getNotificationSettings { settings ->
    when (settings.authorizationStatus) {
        UNAuthorizationStatusAuthorized -> emit(PermissionStatus.GRANTED)
        UNAuthorizationStatusDenied -> emit(PermissionStatus.DENIED)
        UNAuthorizationStatusNotDetermined -> emit(PermissionStatus.NOT_DETERMINED)
        UNAuthorizationStatusProvisional -> emit(PermissionStatus.GRANTED)
        UNAuthorizationStatusEphemeral -> emit(PermissionStatus.GRANTED)
    }
}
```

- **A)** Как `DENIED`
- **B)** Как `NOT_DETERMINED`
- **C)** Как `GRANTED`
- **D)** Эти статусы в коде отсутствуют

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

В сгенерированном `when` оба статуса — `UNAuthorizationStatusProvisional` и `UNAuthorizationStatusEphemeral` — отображаются в `PermissionStatus.GRANTED`. Остальные статусы: Authorized → GRANTED, Denied → DENIED, NotDetermined → NOT_DETERMINED.

**Почему не другие:**
- A — `DENIED` только для `UNAuthorizationStatusDenied`
- B — `NOT_DETERMINED` только для `UNAuthorizationStatusNotDetermined`
- D — оба статуса присутствуют в коде

</details>

---

## Вопрос 8. Какие API используются в примере биометрической аутентификации?

```text
Создай expect/actual для биометрической аутентификации (Face ID / Fingerprint).

Требования:
- Методы: authenticate(prompt, reason), isBiometricAvailable()
- Android: BiometricPrompt API
- iOS: LocalAuthentication (LAContext)

Обработай отмену пользователем и ошибки.
```

- **A)** Android: BiometricPrompt API; iOS: LocalAuthentication (LAContext)
- **B)** Android: FusedLocationProviderClient; iOS: CoreLocation
- **C)** Android: NotificationManager; iOS: UserNotifications
- **D)** Android и iOS: Firebase Cloud Messaging

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Пример 2 темы 2.1 использует `BiometricPrompt` на Android и `LocalAuthentication` (`LAContext`) на iOS, с методами `authenticate(prompt, reason)` и `isBiometricAvailable()`, плюс обработка отмены пользователем и ошибок.

**Почему не другие:**
- B — это API из задания 2.1 (геолокация)
- C — это API из примера с уведомлениями
- D — FCM упоминался только как Android-часть push-уведомлений

</details>

---

## Вопрос 9. Какие платформенные API нужно использовать в задании 2.1 «Location Service»?

- **A)** Android: `FusedLocationProviderClient`; iOS: `CoreLocation`
- **B)** Android: `BiometricPrompt`; iOS: `LAContext`
- **C)** Android: `NotificationManager`; iOS: `UNUserNotificationCenter`
- **D)** Android: `SharedPreferences`; iOS: `UserDefaults`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Задание 2.1: сгенерировать expect/actual для геолокации, на Android — `FusedLocationProviderClient`, на iOS — `CoreLocation`. Критерии: компиляция на обеих платформах и обработка разрешений.

**Почему не другие:**
- B — это биометрия (пример 2)
- C — это уведомления (пример 1)
- D — это хранилища настроек, в задании не упоминаются

</details>

---

## Вопрос 10. Что требует шаблон промпта для Compose UI из модуля?

- **A)** `@Composable`, State hoisting через `remember { mutableStateOf(...) }`, Material 3, тёмная тема через `MaterialTheme.colorScheme`, accessibility (`contentDescription`)
- **B)** XML-разметку и `findViewById`
- **C)** Только `@Preview` без параметров
- **D)** Использование Flutter-виджетов

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаблон для Compose задаёт: `@Composable`-функции, State hoisting с `remember { mutableStateOf(...) }`, компоненты Material 3, поддержку тёмной темы через `MaterialTheme.colorScheme`, двустороннее связывание через `onValueChange`, separation of concerns и accessibility через `contentDescription`. Также просят data class состояния, саму функцию и пример использования в родителе.

**Почему не другие:**
- B — это классический Android View подход, не Compose
- C — превью не заменяет компонент
- D — модуль про Compose Multiplatform, не Flutter

</details>

---

## Вопрос 11. Сколько категорий и каких перечислено в сгенерированном `HabitCategory`?

```kotlin
enum class HabitCategory(
    val name: String,
    val color: String,
    val icon: ImageVector
) {
    TRANSPORT("Транспорт", "#4CAF50", Icons.Default.DirectionsBus),
    WASTE("Мусор", "#FF9800", Icons.Default.Recycling),
    ENERGY("Энергия", "#FFC107", Icons.Default.BrightnessMedium),
    WATER("Вода", "#2196F3", Icons.Default.WaterDrop),
    FOOD("Еда", "#8BC34A", Icons.Default.LocalDining)
}
```

- **A)** 4: TRANSPORT, WASTE, ENERGY, WATER
- **B)** 6: плюс HEALTH
- **C)** 3: TRANSPORT, WASTE, FOOD
- **D)** 5: TRANSPORT, WASTE, ENERGY, WATER, FOOD

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: D**

В коде пять категорий: TRANSPORT, WASTE, ENERGY, WATER, FOOD. Каждая хранит `name` (для `contentDescription`), `color` (hex) и `icon` (`ImageVector`). Это полный код из примера HabitCard.

**Почему не другие:**
- A — пропущена FOOD
- B — HEALTH в перечислении отсутствует
- C — пропущены ENERGY и WATER

</details>

---

## Вопрос 12. Зачем в примере состояние помечено `@Immutable`?

```kotlin
@Immutable
data class HabitCardState(
    val id: String,
    val title: String,
    val category: HabitCategory,
    val streak: Int,
    val isCompleted: Boolean
)
```

- **A)** Чтобы сообщить Compose-компилятору, что объект неизменяем и его можно считать стабильным при рекомпозициях
- **B)** Чтобы запретить чтение полей вне модуля
- **C)** Чтобы автоматически сериализовать объект в JSON
- **D)** Чтобы Kotlin сделал копию объекта при каждом использовании

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`@Immutable` (из `androidx.compose.runtime`) — это подсказка Compose-компилятору: класс неизменяем, все поля `val`, значит его можно считать стабильным и не пересчитывать лишнее при рекомпозиции. Практическая польза — меньше лишних рекомпозиций и предсказуемое поведение UI.

**Почему не другие:**
- B — `val` уже делает поля read-only, аннотация не меняет видимость
- C — для JSON есть `@Serializable`, у модели данных в теме 2.3
- D — копирование — не аннотация, а `copy()` у data class

</details>

---

## Вопрос 13. Как сгенерированный `HabitCard` красит заголовок, если привычка выполнена?

```kotlin
Text(
    text = state.title,
    style = MaterialTheme.typography.titleMedium,
    fontWeight = FontWeight.SemiBold,
    color = if (state.isCompleted) {
        MaterialTheme.colorScheme.onSurfaceVariant.copy(alpha = 0.6f)
    } else {
        MaterialTheme.colorScheme.onSurface
    }
)
```

- **A)** `MaterialTheme.colorScheme.onSurface` всегда
- **B)** `MaterialTheme.colorScheme.onSurfaceVariant.copy(alpha = 0.6f)` — приглушённый цвет
- **C)** `MaterialTheme.colorScheme.primary` — акцентный цвет
- **D)** `MaterialTheme.colorScheme.error`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В коде ветка `isCompleted == true` использует `onSurfaceVariant` с `alpha = 0.6f` — текст становится приглушённым, визуально помечая выполненную привычку. Иначе используется обычный `onSurface`.

**Почему не другие:**
- A — это цвет невыполненного состояния
- C — `primary` используется у Checkbox (`checkedColor`)
- D — `error` в компоненте не встречается

</details>

---

## Вопрос 14. В шаблоне генерации моделей где должна быть аннотация `@Serializable`?

```text
Требования:
1. Domain model (business logic, без аннотаций)
2. Data model (с @Serializable для JSON)
3. Database entity (для SQLDelight)
4. Мappers между слоями
```

- **A)** В `Domain.Habit` — чтобы бизнес-логика знала про JSON
- **B)** В `Db.HabitEntity` — для SQLDelight-запросов
- **C)** В `Data.HabitDto` — модели данных для JSON
- **D)** Во всех трёх моделях одновременно

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

Шаблон прямо разделяет слои: Domain — чистая бизнес-логика **без аннотаций**, Data — `@Serializable` для JSON (`kotlinx.serialization`), Db — entity для SQLDelight, плюс mapper'ы `Domain <-> Data` и `Domain <-> Db`. В примере Habit это 4 класса: `Domain.Habit`, `Data.HabitDto`, `Db.HabitEntity` и функции-мапперы.

**Почему не другие:**
- A — нарушает чистоту Domain-слоя
- B — SQLDelight работает со своими entity, JSON-аннотация тут не нужна
- D — аннотация нужна только в Data-слое

</details>

---

## Вопрос 15. Что возвращает Use Case и какие компоненты создаёт шаблон темы 2.4?

```text
Требования:
- Входные параметры: [ПАРАМЕТРЫ]
- Валидация входных данных
- Бизнес-логика: [ОПИСАНИЕ]
- Возвращает Result<Output, Error>

Обработай ошибки:
- Валидация входных данных
- Ошибки repository
- Бизнес-правила

Создай:
1. Input data class с валидацией
2. Output data class
3. Sealed class Error
4. Use Case класс с execute() методом
```

- **A)** `Result<Output, Error>`; создаёт Input, Output, sealed Error и класс с `execute()`
- **B)** `Boolean`; создаёт только один класс
- **C)** `Flow<String>`; создаёт Repository и DI-модуль
- **D)** `Unit`; ошибочные случаи выбрасывают `Exception` без типизации

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаблон use case из Clean Architecture/Domain: на входе — Input data class с валидацией, на выходе — `Result<Output, Error>`, ошибки трёх видов (валидация, repository, бизнес-правила) описывает sealed class Error, запуск — метод `execute()`. Это позволяет обрабатывать ошибки типизированно.

**Почему не другие:**
- B — булево значение на выходе не передаёт ни данные, ни причину ошибки
- C — `Flow` и Repository/DI здесь не описаны в шаблоне use case
- D — нетипизированные исключения противоречат `Result<Output, Error>`

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 12–15 | 🎉 Отлично! Генерация expect/actual, Compose, моделей и use case'ов — твой уровень |
| 9–11 | 📖 Перечитай темы, где ошибся, и проверь код примера из модуля ещё раз |
| 5–8 | ⚠️ Пройди модуль заново: сначала expect/actual и уведомления, затем Compose и модели |
| < 5 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> ⚠️ Не забывай про раздел «Частые ошибки»: если AI выдаёт устаревшие API — уточни версии (`Kotlin 1.9+`, `Compose 1.5+`); если не учитывает контекст — добавь «В нашем проекте уже используем Koin для DI»; если появляются дубли — попроси «Используй существующие компоненты из ui/components/».
>
> 🔜 Дальше — [Модуль 3: AI для архитектуры и дизайна](../Module_03_AI_Architecture/content.md).

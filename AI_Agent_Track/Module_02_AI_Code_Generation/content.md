# Модуль 2: Генерация KMP кода с AI

## 📋 Обзор модуля

**Продолжительность:** 8 часов  
**Сложность:** Beginner-Intermediate  
**Цель:** Освоение генерации KMP кода (expect/actual, Compose UI, модели данных) с помощью AI

---

## 🎯 Цели обучения

После прохождения модуля вы сможете:
- ✅ Генерировать **expect/actual реализации** для платформенных API
- ✅ Создавать **Compose UI компоненты** с AI
- ✅ Автоматически создавать **модели данных и DTO**
- ✅ Писать **базовую бизнес-логику** с AI-ассистентом

---

## 📚 Темы модуля

### Тема 2.1: Генерация expect/actual реализаций (2 часа)

#### Шаблон prompt'а для expect/actual:

```
Ты - Senior KMP разработчик. Создай expect/actual реализации для [ФУНКЦИОНАЛ].

Контекст:
- Приложение: EcoTrack (трекер эко-привычек)
- Платформы: Android, iOS, Desktop

Требования к expect (commonMain):
- Интерфейс с методами: [СПИСОК МЕТОДОВ]
- Возвращает Flow или suspend функции
- Обработка ошибок через sealed class Result

Требования к actual:
- Android: используй [ANDROID API]
- iOS: используй [iOS FRAMEWORK]
- Desktop: fallback implementation или throw NotImplementedError

Дополнительно:
- Добавь KDoc комментарии
- Обработай edge cases
- Включи логирование через ктлog

Создай 3 файла:
1. commonMain - expect interface и data classes
2. androidMain - actual implementation  
3. iosMain - actual implementation

Покажи полный код каждого файла.
```

---

#### Пример 1: Генерация Notifications API

**Промпт:**
```
Создай expect/actual для push-уведомлений.

Требования:
- Методы: sendNotification(title, body), requestPermission(), getPermissionStatus()
- Android: Firebase Cloud Messaging (FCM)
- iOS: UserNotifications + APNs

Обработай разрешения и ошибки.
```

**Ожидаемый результат от AI:**

**commonMain - Expect Interface:**
```kotlin
// commonMain/platform/NotificationService.kt
package platform

import kotlinx.coroutines.flow.Flow

sealed class NotificationResult {
    data class Success(val notificationId: String) : NotificationResult()
    data class Error(val message: String, val errorCode: Int? = null) : NotificationResult()
}

enum class PermissionStatus {
    GRANTED, DENIED, NOT_DETERMINED, RESTRICTED
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

**androidMain - Actual Implementation:**
```kotlin
// androidMain/platform/NotificationService.android.kt
package platform

import android.app.NotificationChannel
import android.app.NotificationManager
import android.content.Context
import android.os.Build
import androidx.core.app.NotificationCompat
import com.google.firebase.messaging.FirebaseMessaging
import kotlinx.coroutines.flow.*

actual class NotificationService actual constructor(
    private val context: Context
) : NotificationService {
    
    private val notificationManager = 
        context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
    
    init {
        createNotificationChannel()
    }
    
    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                "ecotrack_channel",
                "EcoTrack Notifications",
                NotificationManager.IMPORTANCE_HIGH
            ).apply {
                description = "Уведомления о привычках"
            }
            
            notificationManager.createNotificationChannel(channel)
        }
    }
    
    actual suspend fun sendNotification(
        title: String,
        body: String,
        data: Map<String, String>
    ): NotificationResult {
        return try {
            val notificationId = System.currentTimeMillis().toInt()
            
            val notification = NotificationCompat.Builder(context, "ecotrack_channel")
                .setSmallIcon(android.R.drawable.ic_dialog_info)
                .setContentTitle(title)
                .setContentText(body)
                .setAutoCancel(true)
                .build()
            
            notificationManager.notify(notificationId, notification)
            
            NotificationResult.Success(notificationId.toString())
        } catch (e: Exception) {
            NotificationResult.Error(e.message ?: "Unknown error")
        }
    }
    
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
    
    actual fun cancelNotification(id: Int) {
        notificationManager.cancel(id)
    }
}

actual fun NotificationService(): NotificationService {
    return NotificationService(androidContext())
}
```

**iosMain - Actual Implementation:**
```kotlin
// iosMain/platform/NotificationService.ios.kt
package platform

import kotlinx.coroutines.flow.*
import platform.UserNotifications.*

actual class NotificationService actual constructor() : NotificationService {
    
    private val center = UNUserNotificationCenter.currentCenter()
    
    actual suspend fun sendNotification(
        title: String,
        body: String,
        data: Map<String, String>
    ): NotificationResult {
        return try {
            val content = UNMutableNotificationContent().apply {
                this.title = title
                this.body = body
                // Можно добавить data через userInfo
            }
            
            val request = UNNotificationRequest(
                identifier = "notification_${System.currentTimeMillis()}",
                content = content,
                trigger = null
            )
            
            center.addNotificationRequest(request) { error ->
                if (error != null) {
                    // Обработка ошибки
                }
            }
            
            NotificationResult.Success(request.identifier)
        } catch (e: Exception) {
            NotificationResult.Error(e.message ?: "Unknown error")
        }
    }
    
    actual suspend fun requestPermission(): PermissionStatus {
        return center.requestAuthorization(
            UNAuthorizationOptionBadge or 
            UNAuthorizationOptionSound or 
            UNAuthorizationOptionAlert
        ) { granted, error ->
            // Обработка результата
        }
    }
    
    actual fun getPermissionStatus(): Flow<PermissionStatus> {
        return flow {
            center.getNotificationSettings { settings in
                when (settings.authorizationStatus) {
                    UNAuthorizationStatusAuthorized -> emit(PermissionStatus.GRANTED)
                    UNAuthorizationStatusDenied -> emit(PermissionStatus.DENIED)
                    UNAuthorizationStatusNotDetermined -> emit(PermissionStatus.NOT_DETERMINED)
                    UNAuthorizationStatusProvisional -> emit(PermissionStatus.GRANTED)
                    UNAuthorizationStatusEphemeral -> emit(PermissionStatus.GRANTED)
                }
            }
        }.cacheIn(/* coroutineScope */)
    }
    
    actual fun cancelNotification(id: Int) {
        center.removePendingNotificationRequestsWithIdentifiers(
            listOf("notification_$id")
        )
    }
}

actual fun NotificationService(): NotificationService {
    return NotificationService()
}
```

---

#### Пример 2: Генерация Biometric Authentication

**Промпт:**
```
Создай expect/actual для биометрической аутентификации (Face ID / Fingerprint).

Требования:
- Методы: authenticate(prompt, reason), isBiometricAvailable()
- Android: BiometricPrompt API
- iOS: LocalAuthentication (LAContext)

Обработай отмену пользователем и ошибки.
```

**Ожидаемый результат:** AI сгенерирует полный код для обоих платформ.

---

### Тема 2.2: Генерация Compose UI компонентов (2 часа)

#### Шаблон prompt'а для Compose:

```
Ты - Senior Compose Multiplatform разработчик. Создай UI компонент для [КОМПОНЕНТ].

Контекст:
- Приложение: EcoTrack
- Дизайн-система: Material 3

Требования:
- Используй @Composable функции
- State hoisting с remember { mutableStateOf(...) }
- Material 3 компоненты (Card, Button, TextField и т.д.)
- Поддержка темной темы через MaterialTheme.colorScheme

Функционал:
[ОПИСАНИЕ ФУНКЦИОНАЛА]

Создай:
1. Data class для state (если нужно)
2. @Composable функцию с параметрами
3. Пример использования в parent composable

Следуй best practices:
- Двустороннее связывание через onValueChange
- Separation of concerns (UI логика отдельно от бизнес-логики)
- Accessibility (contentDescription для изображений)

Покажи полный код.
```

---

#### Пример: Создание Habit Card компонента

**Промпт:**
```
Создай Composable компонент HabitCard для отображения привычки.

Требования:
- Поля: title (String), category (HabitCategory), streak (Int), isCompleted (Boolean)
- UI: Card с заголовком, иконкой категории, счетчиком streak, checkbox
- Actions: onClick для completion toggle, onLongClick для edit

Используй Material 3 Card, Checkbox, Icon.
Поддерживай темную тему.

Создай:
1. Data class HabitCardState
2. @Composable HabitCard() с параметрами
3. Enum HabitCategory с иконками

Покажи полный код.
```

**Ожидаемый результат от AI:**

```kotlin
// commonMain/ui/components/HabitCard.kt
package ui.components

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.text.font.FontWeight

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

@Immutable
data class HabitCardState(
    val id: String,
    val title: String,
    val category: HabitCategory,
    val streak: Int,
    val isCompleted: Boolean
)

@Composable
fun HabitCard(
    state: HabitCardState,
    onCompletionToggle: (String) -> Unit,
    onEditClick: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        modifier = modifier
            .fillMaxWidth()
            .padding(vertical = 8.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 2.dp),
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.surfaceVariant
        )
    ) {
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            // Checkbox
            Checkbox(
                checked = state.isCompleted,
                onCheckedChange = { 
                    onCompletionToggle(state.id)
                },
                colors = CheckboxDefaults.colors(
                    checkedColor = MaterialTheme.colorScheme.primary
                )
            )
            
            Spacer(modifier = Modifier.width(12.dp))
            
            // Category icon
            Icon(
                imageVector = state.category.icon,
                contentDescription = state.category.name,
                tint = MaterialTheme.colorScheme.secondary
            )
            
            Spacer(modifier = Modifier.width(12.dp))
            
            // Title and streak
            Column {
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
                
                Row(
                    horizontalArrangement = Arrangement.spacedBy(4.dp),
                    verticalAlignment = Alignment.Bottom
                ) {
                    Text(
                        text = "🔥 ${state.streak} дней",
                        style = MaterialTheme.typography.bodySmall,
                        color = MaterialTheme.colorScheme.secondary
                    )
                }
            }
            
            Spacer(modifier = Modifier.weight(1f))
            
            // Edit button (visible on long press)
            IconButton(onClick = { onEditClick(state.id) }) {
                Icon(
                    imageVector = Icons.Default.Edit,
                    contentDescription = "Edit habit",
                    tint = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
        }
    }
}
```

---

### Тема 2.3: Генерация моделей данных и DTO (1.5 часа)

#### Шаблон prompt'а для моделей:

```
Ты - Senior KMP разработчик. Создай модели данных для [СУЩНОСТЬ].

Контекст:
- Приложение: EcoTrack
- Используем kotlinx.serialization для JSON
- SQLDelight для persistence

Требования:
1. Domain model (business logic, без аннотаций)
2. Data model (с @Serializable для JSON)
3. Database entity (для SQLDelight)
4. Мappers между слоями

Сущность: [НАЗВАНИЕ] с полями:
- id: UUID
- title: String (не пустой)
- category: [ENUM]
- createdAt: Instant
- updatedAt: Instant
- streak: Int (>= 0)
- isCompleted: Boolean

Добавь валидацию и factory methods.
Покажи полный код всех 4 типов моделей + mappers.
```

---

#### Пример: Генерация модели Habit

**Промпт:**
```
Создай полную иерархию моделей для Habit (привычка).

Поля:
- id: UUID
- title: String (2-100 символов)
- category: HabitCategory enum
- description: String? (опционально, до 500 символов)
- targetPerWeek: Int (1-7)
- currentWeekCount: Int (0-7)
- streak: Int (>= 0)
- createdAt: Instant
- updatedAt: Instant

Создай:
1. Domain.Habit (чистая бизнес-логика)
2. Data.HabitDto (@Serializable)
3. Db.HabitEntity (для SQLDelight queries)
4. Mappers: Domain <-> Data, Domain <-> Db

Добавь валидацию и factory methods.
```

**Ожидаемый результат:** AI сгенерирует 4 класса моделей + mapper функции.

---

### Тема 2.4: Генерация бизнес-логики (1.5 часа)

#### Шаблон prompt'а для use cases:

```
Ты - Senior KMP разработчик. Создай Use Case для [ДЕЙСТВИЕ].

Контекст:
- Clean Architecture (Domain слой)
- Repository pattern

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

Покажи полный код.
```

---

## 📝 Практические задания модуля

### Задание 2.1: Создать Location Service с AI (2 часа)

**Требования:**
- Генерируй expect/actual для геолокации
- Android: FusedLocationProviderClient
- iOS: CoreLocation

**Критерии:**
- ✓ Код компилируется на обеих платформах
- ✓ Обработка разрешений

---

### Задание 2.2: Создать UI компоненты для EcoTrack (3 часа)

**Требования:**
- HabitCard, CategorySelector, StreakCounter
- Используй Material 3

**Критерии:**
- ✓ Компоненты переиспользуемые
- ✓ Поддержка темной темы

---

### Задание 2.3: Создать полную иерархию моделей (2 часа)

**Требования:**
- Domain, Data, Db модели для Habit и Category
- Mappers между слоями

**Критерии:**
- ✓ 80%+ кода сгенерировано AI
- ✓ Код компилируется и проходит тесты

---

## 🚫 Частые ошибки при генерации кода с AI

### ❌ AI генерирует устаревшие API
**Решение:** Указывайте версии в prompt'е: "Используй Kotlin 1.9+, Compose 1.5+"

### ❌ AI не учитывает ваш контекст
**Решение:** Добавляйте контекст: "В нашем проекте уже используем Koin для DI"

### ❌ AI создает code duplication
**Решение:** Просите: "Используй существующие компоненты из ui/components/"

---

## 📚 Дополнительные материалы

### Prompt библиотеки:
- [KMP Prompt Library](https://github.com/kmp-prompts) - 100+ prompt'ов для KMP
- [Compose Prompt Collection](https://github.com/compose-prompts) - UI компоненты

### Инструменты:
- [Kotlin Playground](https://pl.kotl.in/) - Тестирование сгенерированного кода
- [Compose Preview](https://developer.android.com/jetpack/compose/preview) - Визуализация UI

---

## 🚀 Следующий шаг

Переходите к [Модулю 3](../Module_03_AI_Architecture/content.md): AI для архитектуры и дизайна

**Время до следующего модуля:** 1-2 недели  
**Рекомендуемая практика:** Генерируйте минимум 30 строк кода в день с AI

---

**Удачи в генерации кода с помощью ИИ! 🤖💻**

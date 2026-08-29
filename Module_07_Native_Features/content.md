# 📘 Модуль 7: Нативные фичи и Интеграции

**Добро пожаловать в седьмой модуль!**  
Теперь, когда приложение готово к релизу, пришло время добавить нативные фичи: push-уведомления, deep links и работу с файловой системой. Мы научимся правильно использовать Expect/Actual для платформенных зависимостей.

**Цели модуля:**
1. Реализовать push-уведомления (FCM для Android, APNs для iOS)
2. Настроить deep links для открытия конкретных экранов
3. Работать с файловой системой (экспорт/импорт данных)
4. Интегрировать аналитику и краш-репорты

**Время выполнения:** ~12–15 часов.

---

## 1. Теория: Expect/Actual для нативных фич

### Паттерн Expect/Actual

```kotlin
// commonMain - объявляем интерфейс (контракт)
expect fun sendNotification(title: String, body: String): Result<Unit>

// androidMain - реализация для Android
actual fun sendNotification(title: String, body: String): Result<Unit> {
    // FCM / Firebase Cloud Messaging
}

// iosMain - реализация для iOS  
actual fun sendNotification(title: String, body: String): Result<Unit> {
    // APNs / UserNotifications
}
```

**Правила:**
1. **Expect** — только в commonMain, объявляет контракт
2. **Actual** — реализация для каждой платформы
3. **Синхронизация:** сигнатуры должны совпадать

---

## 2. Практика: Push-уведомления

### Шаг 1: Создаем интерфейс уведомлений

**Файл:** `src/commonMain/kotlin/com/ecotrack/platform/NotificationManager.kt`

```kotlin
package com.ecotrack.platform

// Контракт для уведомлений (общий код)
interface NotificationManager {
    suspend fun requestPermission(): Result<Boolean>
    suspend fun sendNotification(
        title: String, 
        body: String,
        data: Map<String, String>? = null
    ): Result<Unit>
    
    fun isPermissionGranted(): Boolean
}

// Ожидаем реализацию от платформы
expect fun getNotificationManager(): NotificationManager
```

### Шаг 2: Реализация для Android (FCM)

**Файл:** `src/androidMain/kotlin/com/ecotrack/platform/NotificationManager.kt`

```kotlin
package com.ecotrack.platform

import android.Manifest
import android.app.NotificationChannel
import android.app.NotificationManager
import android.content.Context
import android.os.Build
import androidx.core.app.NotificationCompat
import com.google.firebase.messaging.FirebaseMessaging
import kotlinx.coroutines.tasks.await

class AndroidNotificationManager(
    private val context: Context
) : NotificationManager {
    
    companion object {
        const val CHANNEL_ID = "ecotrack_channel"
        const val CHANNEL_NAME = "EcoTrack Notifications"
    }
    
    init {
        createNotificationChannel()
    }
    
    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                CHANNEL_ID, 
                CHANNEL_NAME,
                NotificationManager.IMPORTANCE_HIGH
            ).apply {
                description = "Уведомления о привычках"
            }
            
            val notificationManager = context.getSystemService(
                Context.NOTIFICATION_SERVICE
            ) as NotificationManager
            
            notificationManager.createNotificationChannel(channel)
        }
    }
    
    override suspend fun requestPermission(): Result<Boolean> {
        return try {
            // Для Android 13+ нужно запрашивать разрешение
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                val granted = checkPermission()
                Result.success(granted)
            } else {
                // Android 12 и ниже - разрешения не нужны
                Result.success(true)
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    private fun checkPermission(): Boolean {
        // Реализация проверки разрешения (упрощенно)
        return true
    }
    
    override suspend fun sendNotification(
        title: String, 
        body: String,
        data: Map<String, String>?
    ): Result<Unit> = try {
        val notification = NotificationCompat.Builder(context, CHANNEL_ID)
            .setSmallIcon(android.R.drawable.ic_dialog_info) // Замените на ваш иконку
            .setContentTitle(title)
            .setContentText(body)
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setAutoCancel(true)
            .build()
        
        val notificationManager = context.getSystemService(
            Context.NOTIFICATION_SERVICE
        ) as NotificationManager
        
        notificationManager.notify(System.currentTimeMillis().toInt(), notification)
        
        Result.success(Unit)
    } catch (e: Exception) {
        Result.failure(e)
    }
    
    override fun isPermissionGranted(): Boolean = checkPermission()
    
    // Получение FCM токена для сервера
    suspend fun getFcmToken(): Result<String> = try {
        val token = FirebaseMessaging.getInstance().token.await()
        Result.success(token)
    } catch (e: Exception) {
        Result.failure(e)
    }
}

actual fun getNotificationManager(): NotificationManager {
    // В реальном приложении получаем context из Application
    return AndroidNotificationManager(/* context */)
}
```

### Шаг 3: Реализация для iOS (APNs)

**Файл:** `src/iosMain/kotlin/com/ecotrack/platform/NotificationManager.kt`

```kotlin
package com.ecotrack.platform

import platform.UserNotifications.UNAuthorizationOptions
import platform.UserNotifications.UNAuthorizationOptionAlert
import platform.UserNotifications.UNAuthorizationOptionBadge
import platform.UserNotifications.UNAuthorizationOptionSound
import platform.UserNotifications.UNMutableNotificationContent
import platform.UserNotifications.UNNotificationRequest
import platform.UserNotifications.UNTimeIntervalSinceEpoch
import platform.darwin.NSUUID
import kotlinx.cinterop.ExperimentalForeignApi
import kotlinx.coroutines.suspendCancellableCoroutine
import kotlin.coroutines.resume

class IOSNotificationManager : NotificationManager {
    
    @OptIn(ExperimentalForeignApi::class)
    override suspend fun requestPermission(): Result<Boolean> = 
        suspendCancellableCoroutine { continuation ->
            val center = platform.UserNotifications.UNUserNotificationCenter.currentNotificationCenter()
            
            center.requestAuthorizationWithOptions(
                UNAuthorizationOptionAlert or 
                UNAuthorizationOptionBadge or 
                UNAuthorizationOptionSound
            ) { granted, error ->
                if (error != null) {
                    continuation.resume(Result.failure(Exception(error.localizedDescription)))
                } else {
                    continuation.resume(Result.success(granted))
                }
            }
        }
    
    @OptIn(ExperimentalForeignApi::class)
    override suspend fun sendNotification(
        title: String, 
        body: String,
        data: Map<String, String>?
    ): Result<Unit> = try {
        val content = UNMutableNotificationContent().apply {
            title = title
            body = body
        }
        
        val trigger = platform.UserNotifications.UNTimeIntervalTrigger.timeIntervalTriggerWithTimeInterval(0.0)
        val request = UNNotificationRequest.requestWithIdentifier(
            NSUUID().UUIDString,
            content,
            trigger
        )
        
        val center = platform.UserNotifications.UNUserNotificationCenter.currentNotificationCenter()
        center.addNotificationRequest(request) { error ->
            // Уведомление отправлено
        }
        
        Result.success(Unit)
    } catch (e: Exception) {
        Result.failure(e)
    }
    
    override fun isPermissionGranted(): Boolean = 
        platform.UserNotifications.UNUserNotificationCenter.currentNotificationCenter()
            .getNotificationSettingsWithCompletionHandler { settings ->
                // Проверка настроек (упрощенно)
            }
    
    // Получение APNs токена для сервера
    suspend fun getApnsToken(): Result<String> {
        // Реализация получения APNs токена (сложная, требует настройки)
        return Result.failure(Exception("APNs token retrieval not implemented"))
    }
}

actual fun getNotificationManager(): NotificationManager = IOSNotificationManager()
```

### Шаг 4: Использование в ViewModel

**Файл:** `src/commonMain/kotlin/com/ecotrack/ui/viewmodels/HabitViewModel.kt`

```kotlin
package com.ecotrack.ui.viewmodels

import androidx.lifecycle.ViewModel
import com.ecotrack.domain.model.HabitEntity
import com.ecotrack.domain.repository.HabitRepository
import com.ecotrack.platform.getNotificationManager
import kotlinx.coroutines.flow.*

class HabitViewModel(
    private val repository: HabitRepository,
    private val notificationManager = getNotificationManager() // Получаем из expect/actual
) : ViewModel() {
    
    init {
        requestNotificationPermission()
    }
    
    private fun requestNotificationPermission() {
        viewModelScope.launch {
            when (val result = notificationManager.requestPermission()) {
                is Result.Success -> {
                    if (result.getOrNull() == true) {
                        // Разрешение получено, можно отправлять уведомления
                    }
                }
                is Result.Failure -> {
                    // Обработка ошибки
                }
            }
        }
    }
    
    fun addHabit(title: String, category: HabitCategory) {
        viewModelScope.launch {
            // ... логика добавления
            
            // Отправляем уведомление об успехе
            if (notificationManager.isPermissionGranted()) {
                notificationManager.sendNotification(
                    title = "Привычка добавлена!",
                    body = "$title успешно сохранена"
                )
            }
        }
    }
    
    // Ежедневное напоминание (через WorkManager для Android / BackgroundTasks для iOS)
    fun scheduleDailyReminder() {
        // Реализация зависит от платформы (в модуле 8)
    }
}
```

---

## 3. Практика: Deep Links

### Шаг 1: Создаем интерфейс для deep links

**Файл:** `src/commonMain/kotlin/com/ecotrack/platform/DeepLinkManager.kt`

```kotlin
package com.ecotrack.platform

// Контракт для deep links
interface DeepLinkManager {
    fun registerDeepLinks()
    suspend fun handleDeepLink(link: String): Result<DeepLinkAction>
}

enum class DeepLinkAction {
    OPEN_HABIT,      // Открыть конкретную привычку
    ADD_HABIT,       // Добавить новую привычку
    VIEW_STATISTICS  // Открыть статистику
}

expect fun getDeepLinkManager(): DeepLinkManager
```

### Шаг 2: Реализация для Android

**Файл:** `src/androidMain/kotlin/com/ecotrack/platform/DeepLinkManager.kt`

```kotlin
package com.ecotrack.platform

import android.content.Intent
import androidx.core.app.TaskStackBuilder

class AndroidDeepLinkManager(
    private val context: Context
) : DeepLinkManager {
    
    override fun registerDeepLinks() {
        // Настройка в AndroidManifest.xml (см. ниже)
    }
    
    override suspend fun handleDeepLink(link: String): Result<DeepLinkAction> {
        return try {
            val uri = Uri.parse(link)
            
            when (uri.path) {
                "/habit" -> Result.success(DeepLinkAction.OPEN_HABIT)
                "/add-habit" -> Result.success(DeepLinkAction.ADD_HABIT)
                "/statistics" -> Result.success(DeepLinkAction.VIEW_STATISTICS)
                else -> Result.failure(Exception("Неизвестный deep link"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

actual fun getDeepLinkManager(): DeepLinkManager = AndroidDeepLinkManager(/* context */)
```

**Файл:** `src/androidMain/AndroidManifest.xml`

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application>
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
            
            <!-- Deep links -->
            <intent-filter>
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />
                
                <!-- HTTP scheme -->
                <data android:scheme="https" android:host="ecotrack.app" />
                <data android:pathPrefix="/habit" />
                <data android:pathPrefix="/add-habit" />
                <data android:pathPrefix="/statistics" />
            </intent-filter>
            
            <!-- Custom scheme -->
            <intent-filter>
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />
                
                <data android:scheme="ecotrack" android:host="habit" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

### Шаг 3: Реализация для iOS

**Файл:** `src/iosMain/kotlin/com/ecotrack/platform/DeepLinkManager.kt`

```kotlin
package com.ecotrack.platform

class IOSDeepLinkManager : DeepLinkManager {
    
    override fun registerDeepLinks() {
        // Настройка в Info.plist (см. ниже)
    }
    
    override suspend fun handleDeepLink(link: String): Result<DeepLinkAction> {
        return try {
            val url = URL(link) ?: throw Exception("Invalid URL")
            
            when (url.path?.removePrefix("/")) {
                "habit" -> Result.success(DeepLinkAction.OPEN_HABIT)
                "add-habit" -> Result.success(DeepLinkAction.ADD_HABIT)
                "statistics" -> Result.success(DeepLinkAction.VIEW_STATISTICS)
                else -> Result.failure(Exception("Неизвестный deep link"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

actual fun getDeepLinkManager(): DeepLinkManager = IOSDeepLinkManager()
```

**Файл:** `src/iosMain/Info.plist`

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>ecotrack</string>
        </array>
        <key>CFBundleURLName</key>
        <string>com.ecotrack.app</string>
    </dict>
</array>

<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

---

## 4. Практика: Работа с файловой системой

### Шаг 1: Создаем интерфейс для файлов

**Файл:** `src/commonMain/kotlin/com/ecotrack/platform/FileManager.kt`

```kotlin
package com.ecotrack.platform

// Контракт для работы с файлами
interface FileManager {
    suspend fun exportData(filename: String): Result<String>  // Возвращает путь к файлу
    suspend fun importData(filepath: String): Result<Unit>
    suspend fun deleteFile(filepath: String): Result<Unit>
}

expect fun getFileManager(): FileManager
```

### Шаг 2: Реализация для Android

**Файл:** `src/androidMain/kotlin/com/ecotrack/platform/FileManager.kt`

```kotlin
package com.ecotrack.platform

import android.content.Context
import android.content.Intent
import androidx.core.content.FileProvider
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext

class AndroidFileManager(
    private val context: Context
) : FileManager {
    
    override suspend fun exportData(filename: String): Result<String> = 
        withContext(Dispatchers.IO) {
            try {
                val file = File(context.filesDir, filename)
                
                // Записываем данные в файл (JSON или SQLite dump)
                file.writeText(/* serialized data */)
                
                // Создаем URI для FileProvider
                val uri = FileProvider.getUriForFile(
                    context, 
                    "${context.packageName}.fileprovider",
                    file
                )
                
                Result.success(uri.toString())
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    
    override suspend fun importData(filepath: String): Result<Unit> = 
        withContext(Dispatchers.IO) {
            try {
                val file = File(filepath)
                val data = file.readText()
                
                // Парсим и загружаем данные в БД
                
                Result.success(Unit)
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    
    override suspend fun deleteFile(filepath: String): Result<Unit> = 
        withContext(Dispatchers.IO) {
            try {
                File(filepath).delete()
                Result.success(Unit)
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
}

actual fun getFileManager(): FileManager = AndroidFileManager(/* context */)
```

**Файл:** `src/androidMain/AndroidManifest.xml` (добавляем FileProvider)

```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

**Файл:** `src/androidMain/res/xml/file_paths.xml`

```xml
<paths>
    <files-path name="export" path="." />
</paths>
```

### Шаг 3: Реализация для iOS

**Файл:** `src/iosMain/kotlin/com/ecotrack/platform/FileManager.kt`

```kotlin
package com.ecotrack.platform

import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import platform.Foundation.NSDocumentDirectory
import platform.Foundation.NSFileManager
import platform.Foundation.NSURL

class IOSFileManager : FileManager {
    
    private val documentsDirectory: NSURL? by lazy {
        NSFileManager.defaultManager.URLForDirectory(
            NSDocumentDirectory,
            NSDomainMask.NSUserDomainMask,
            null,
            false
        )
    }
    
    override suspend fun exportData(filename: String): Result<String> = 
        withContext(Dispatchers.IO) {
            try {
                val fileUrl = documentsDirectory?.filePath?.let { 
                    "$it/$filename" 
                } ?: throw Exception("Documents directory not found")
                
                // Записываем данные в файл
                File(fileUrl).writeText(/* serialized data */)
                
                Result.success(fileUrl)
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    
    override suspend fun importData(filepath: String): Result<Unit> = 
        withContext(Dispatchers.IO) {
            try {
                val data = File(filepath).readText()
                
                // Парсим и загружаем данные в БД
                
                Result.success(Unit)
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    
    override suspend fun deleteFile(filepath: String): Result<Unit> = 
        withContext(Dispatchers.IO) {
            try {
                File(filepath).delete()
                Result.success(Unit)
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
}

actual fun getFileManager(): FileManager = IOSFileManager()
```

---

## 📝 Домашнее задание (Модуль 7)

### Задание 1: Реализуйте push-уведомления
- Настройте Firebase Cloud Messaging для Android
- Настройте APNs для iOS (требует Apple Developer аккаунта)
- Отправляйте уведомления при добавлении/завершении привычки

### Задание 2: Настройте deep links
- Создайте ссылки вида `ecotrack://habit/{id}` и `https://ecotrack.app/habit/{id}`
- При открытии ссылки открывайте соответствующий экран

### Задание 3: Реализуйте экспорт/импорт данных
- Экспорт в JSON файл с возможностью поделиться (Android Share Sheet / iOS ActivityViewController)
- Импорт из файла для восстановления данных

### Задание 4: Добавьте аналитику
- Интегрируйте Firebase Analytics или Mixpanel через Expect/Actual
- Отслеживайте ключевые события: добавление привычки, завершение, синхронизация

### Задание 5: Настройте краш-репорты
- Интегрируйте Firebase Crashlytics или Sentry через Expect/Actual

**Критерий сдачи:**
- Уведомления приходят корректно на обеих платформах
- Deep links открывают правильные экраны
- Экспорт/импорт работает без потерь данных

---

**💡 Совет:** Для тестирования push-уведомлений на iOS используйте **Development Certificate** и **Provisioning Profile**. Для продакшена потребуется **Apple Developer Program** ($99/год).

**Важно:** Не забудьте добавить разрешения в манифесты:
- Android: `POST_NOTIFICATIONS` (Android 13+)
- iOS: Настройка в Xcode под Signing & Capabilities

Удачи! В следующем (финальном) модуле мы подготовим приложение к релизу: тестирование, оптимизация и публикация в App Store / Google Play.

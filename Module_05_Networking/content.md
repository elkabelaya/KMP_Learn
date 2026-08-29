# 📘 Модуль 5: Сетевое взаимодействие (Networking)

**Добро пожаловать в пятый модуль!**  
Теперь, когда данные сохраняются локально, пришло время сделать их доступными на разных устройствах. Мы подключим **Ktor Client** для работы с API и реализуем синхронизацию данных.

**Цели модуля:**
1. Настроить Ktor Client для KMP (Expect/Actual)
2. Освоить Kotlinx Serialization для JSON-маппинга
3. Реализовать REST API клиент с обработкой ошибок
4. Внедрить Offline-First подход (сначала БД, потом сеть)

**Время выполнения:** ~12–15 часов.

---

## 1. Теория: Networking в KMP

### Варианты HTTP-клиентов для KMP:

| Технология | Плюсы | Минусы | Рекомендация |
|------------|-------|--------|--------------|
| **Ktor Client** | Полная KMP-поддержка, единый API | Нужно настраивать Expect/Actual | ✅ **Рекомендуется** |
| OkHttp Multiplatform | Знакомый API (как на Android) | Меньше фич для iOS | Для простых проектов |
| URLSession (iOS) + OkHttp (Android) | Нативные API | Дублирование кода | ❌ Не рекомендуется |

### Почему Ktor Client?
- **Единый API:** Один код для всех платформ
- **Интерцепторы:** Логирование, авторизация, retry
- **Асинхронность:** Native support для Coroutines
- **JSON:** Встроенная поддержка Kotlinx Serialization

---

## 2. Теория: Expect/Actual для Ktor Client

Ktor Client требует настройки платформенных движков:

```kotlin
// commonMain - объявляем клиент
expect fun createHttpClient(): HttpClient

// androidMain - OkHttp engine
actual fun createHttpClient() = HttpClient(Android) { ... }

// iosMain - iOS engine  
actual fun createHttpClient() = HttpClient(Ios) { ... }
```

---

## 3. Практика: Настройка Ktor Client

### Шаг 1: Добавляем зависимости

**Файл:** `shared/build.gradle.kts` (или `app/build.gradle.kts`)

```kotlin
dependencies {
    // Ktor Client для commonMain
    implementation("io.ktor:ktor-client-core:2.3.7")
    implementation("io.ktor:ktor-client-content-negotiation:2.3.7")
    implementation("io.ktor:ktor-serialization-kotlinx-json:2.3.7")
    
    // Ktor для Android (OkHttp)
    androidImplementation("io.ktor:ktor-client-okhttp:2.3.7")
    
    // Ktor для iOS (Native)
    iosX64Implementation("io.ktor:ktor-client-ios:2.3.7")
    iosArm64Implementation("io.ktor:ktor-client-ios:2.3.7")
    iosSimulatorArm64Implementation("io.ktor:ktor-client-ios:2.3.7")
    
    // Kotlinx Serialization
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.2")
}

// Включение плагина serialization в kotlin { } блок
kotlin {
    // ... existing targets
    
    sourceSets {
        commonMain.dependencies {
            implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.2")
        }
    }
}
```

**Файл:** `settings.gradle.kts` (добавьте плагин)

```kotlin
pluginManagement {
    repositories {
        google()
        gradlePluginPortal()
        maven("https://maven.pkg.jetbrains.space/public/p/compose/dev")
    }
}

plugins {
    id("org.jetbrains.kotlin.plugin.serialization") version "2.0.0"
}
```

### Шаг 2: Создаем DTO (Data Transfer Objects)

**Файл:** `src/commonMain/kotlin/com/ecotrack/data/network/dto/HabitDto.kt`

```kotlin
package com.ecotrack.data.network.dto

import kotlinx.serialization.Serializable

// DTO для JSON-серIALIZАЦИИ
@Serializable
data class HabitDto(
    val id: String,
    val title: String,
    val category: Int,  // Используем int для JSON (enum не сериализуется напрямую)
    val isCompleted: Boolean,
    val createdAt: Long,
    val updatedAt: Long
)

@Serializable
data class HabitResponse(
    val success: Boolean,
    val data: HabitDto? = null,
    val error: String? = null
)

@Serializable
data class HabitsListResponse(
    val success: Boolean,
    val data: List<HabitDto> = emptyList(),
    val error: String? = null
)

@Serializable  
data class AuthToken(
    val token: String,
    val expiresIn: Long
)
```

### Шаг 3: Создаем Expect/Actual для HttpClient

**Файл:** `src/commonMain/kotlin/com/ecotrack/data/network/HttpClientModule.kt`

```kotlin
package com.ecotrack.data.network

import io.ktor.client.*
import io.ktor.client.call.*
import io.ktor.client.request.*
import io.ktor.serialization.kotlinx.json.*

// Ожидаем создание клиента от платформы
expect fun createHttpClient(): HttpClient
```

**Файл:** `src/androidMain/kotlin/com/ecotrack/data/network/HttpClientModule.kt`

```kotlin
package com.ecotrack.data.network

import io.ktor.client.*
import io.ktor.client.engine.okhttp.*
import io.ktor.serialization.kotlinx.json.*

actual fun createHttpClient(): HttpClient = HttpClient(OkHttp) {
    // Настройка для Android
    install(io.ktor.serialization.kotlinx.json.Json) {
        ignoreUnknownKeys = true
    }
    
    // Таймауты
    engine {
        connectTimeoutMs = 15_000
        readTimeoutMs = 30_000
    }
}
```

**Файл:** `src/iosMain/kotlin/com/ecotrack/data/network/HttpClientModule.kt`

```kotlin
package com.ecotrack.data.network

import io.ktor.client.*
import io.ktor.client.engine.darwin.*
import io.ktor.serialization.kotlinx.json.*

actual fun createHttpClient(): HttpClient = HttpClient(Darwin) {
    // Настройка для iOS
    install(io.ktor.serialization.kotlinx.json.Json) {
        ignoreUnknownKeys = true
    }
    
    engine {
        connectTimeoutMs = 15_000
        readTimeoutMs = 30_000
    }
}
```

### Шаг 4: Создаем API Repository

**Файл:** `src/commonMain/kotlin/com/ecotrack/data/network/HabitApiRepository.kt`

```kotlin
package com.ecotrack.data.network

import com.ecotrack.data.network.dto.HabitDto
import com.ecotrack.data.network.dto.HabitsListResponse
import io.ktor.client.*
import io.ktor.client.call.*
import io.ktor.client.request.*
import io.ktor.http.*

// Интерфейс API (общий код)
interface HabitApiRepository {
    suspend fun getHabits(): Result<List<HabitDto>>
    suspend fun createHabit(habit: HabitDto): Result<HabitDto>
    suspend fun updateHabit(id: String, habit: HabitDto): Result<HabitDto>
    suspend fun deleteHabit(id: String): Result<Unit>
}

// Реализация API Repository
class HabitApiRepositoryImpl(
    private val client: HttpClient,
    private val baseUrl: String = "https://api.ecotrack.app" // Замените на ваш API
) : HabitApiRepository {
    
    override suspend fun getHabits(): Result<List<HabitDto>> = safeApiCall {
        val response = client.get("$baseUrl/habits")
        response.body<HabitsListResponse>()
    }.map { it.data }
    
    override suspend fun createHabit(habit: HabitDto): Result<HabitDto> = safeApiCall {
        val response = client.post("$baseUrl/habits") {
            setBody(habit)
        }
        response.body<HabitResponse>()
    }.map { it.data!! }
    
    override suspend fun updateHabit(id: String, habit: HabitDto): Result<HabitDto> = safeApiCall {
        val response = client.put("$baseUrl/habits/$id") {
            setBody(habit)
        }
        response.body<HabitResponse>()
    }.map { it.data!! }
    
    override suspend fun deleteHabit(id: String): Result<Unit> = safeApiCall {
        client.delete("$baseUrl/habits/$id")
        Unit
    }
    
    // Helper для безопасных API-вызовов с обработкой ошибок
    private inline fun <T> safeApiCall(block: () -> T): Result<T> {
        return try {
            val result = block()
            Result.success(result)
        } catch (e: io.ktor.client.network.SocketException) {
            Result.failure(NetworkException("Нет интернета"))
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// Кастомное исключение для сети
class NetworkException(message: String) : Exception(message)
```

### Шаг 5: Создаем Sync Repository (Offline-First)

**Файл:** `src/commonMain/kotlin/com/ecotrack/data/repository/HabitSyncRepository.kt`

```kotlin
package com.ecotrack.data.repository

import com.ecotrack.data.local.HabitRepositoryImpl
import com.ecotrack.data.network.HabitApiRepository
import com.ecotrack.domain.model.HabitEntity
import com.ecotrack.domain.repository.HabitRepository
import kotlinx.coroutines.flow.Flow

// Репозиторий с синхронизацией (Offline-First)
class HabitSyncRepository(
    private val localRepo: HabitRepositoryImpl,  // БД
    private val apiRepo: HabitApiRepository      // Сеть
) : HabitRepository {
    
    override fun getHabits(): Flow<List<HabitEntity>> {
        // Сначала возвращаем из БД (мгновенно)
        return localRepo.getHabits()
    }
    
    override suspend fun addHabit(habit: HabitEntity): Result<Unit> {
        // 1. Сначала сохраняем локально (мгновенно)
        val localResult = localRepo.addHabit(habit)
        
        if (localResult.isFailure) {
            return localResult.map { }
        }
        
        // 2. Затем синхронизируем с сервером (в фоне)
        val dto = habit.toDto()
        apiRepo.createHabit(dto).onFailure { 
            // Логирование ошибки, но не прерываем выполнение
        }
        
        return localResult.map { }
    }
    
    override suspend fun updateHabit(habit: HabitEntity): Result<Unit> {
        val localResult = localRepo.updateHabit(habit)
        
        if (localResult.isFailure) {
            return localResult.map { }
        }
        
        val dto = habit.toDto()
        apiRepo.updateHabit(habit.id, dto).onFailure { }
        
        return localResult.map { }
    }
    
    override suspend fun deleteHabit(id: String): Result<Unit> {
        val localResult = localRepo.deleteHabit(id)
        
        if (localResult.isFailure) {
            return localResult
        }
        
        apiRepo.deleteHabit(id).onFailure { }
        
        return localResult
    }
    
    override fun getHabitsByCategory(category: com.ecotrack.domain.model.HabitCategory): Flow<List<HabitEntity>> {
        return localRepo.getHabitsByCategory(category)
    }
    
    // Синхронизация всех данных с сервера
    suspend fun syncWithServer(): Result<Unit> {
        return try {
            val remoteHabits = apiRepo.getHabits()
            
            if (remoteHabits.isSuccess) {
                // Очищаем локальную БД и загружаем с сервера
                remoteHabits.getOrNull()?.forEach { dto ->
                    localRepo.addHabit(dto.toDomainEntity())
                }
            }
            
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// Extension functions для маппинга
private fun HabitEntity.toDto(): com.ecotrack.data.network.dto.HabitDto = 
    com.ecotrack.data.network.dto.HabitDto(
        id = this.id,
        title = this.title,
        category = this.category.ordinal,
        isCompleted = this.isCompleted,
        createdAt = this.createdAt,
        updatedAt = System.currentTimeMillis()
    )

private fun com.ecotrack.data.network.dto.HabitDto.toDomainEntity(): HabitEntity = 
    HabitEntity(
        id = this.id,
        title = this.title,
        category = com.ecotrack.domain.model.HabitCategory.entries[this.category],
        isCompleted = this.isCompleted,
        createdAt = this.createdAt
    )
```

### Шаг 6: Добавляем интерцепторы (Auth, Logging)

**Файл:** `src/commonMain/kotlin/com/ecotrack/data/network/HttpInterceptors.kt`

```kotlin
package com.ecotrack.data.network

import io.ktor.client.*
import io.ktor.client.call.*
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.http.*

// Интерцептор для логирования всех запросов
fun HttpClient.Builder.addLoggingInterceptor() {
    install(io.ktor.client.plugins.Logged) {
        logger = io.ktor.utils.io.logging.SLF4JLogger("EcoTrack API")
        level = io.ktor.utils.io.logging.LogLevel.BODY
    }
}

// Интерцептор для авторизации (JWT token)
class AuthInterceptor(
    private val getToken: () -> String?  // Функция для получения токена
) : io.ktor.client.plugins.Plugin<Unit, AuthInterceptor> {
    
    companion object : Plugin<Unit, AuthInterceptor>() {
        override val name: String = "AuthInterceptor"
        
        override fun prepare(block: AuthInterceptor.() -> Unit) = AuthInterceptor().apply(block)
        
        override fun install(pipeline: io.ktor.client.HttpClientPipeline, configure: AuthInterceptor.() -> Unit) {
            val interceptor = prepare(configure)
            
            pipeline.intercept(io.ktor.client.request.HttpRequestPipeline.Before) {
                val token = interceptor.getToken()
                
                if (token != null && context.headers["Authorization"] == null) {
                    context.headers.append("Authorization", "Bearer $token")
                }
            }
        }
    }
}

// Интерцептор для retry (повторные попытки)
fun HttpClient.Builder.addRetryInterceptor(maxRetries: Int = 3) {
    install(io.ktor.client.plugins.Retry) {
        maxRetriesCount = maxRetries
        retryOnServerErrors = true
        
        // Retry только для 5xx ошибок, не для 4xx
        retryOnException { request, exception ->
            when (val response = request.response) {
                is io.ktor.client.statement.HttpResponse -> {
                    response.status.value in 500..599
                }
                else -> false
            }
        }
    }
}
```

### Шаг 7: Обновляем Koin модули

**Файл:** `src/commonMain/kotlin/com/ecotrack/data/network/NetworkModule.kt`

```kotlin
package com.ecotrack.data.network

import org.koin.dsl.module

val networkModule = module {
    // HttpClient
    single { createHttpClient().apply {
        addLoggingInterceptor()
        addRetryInterceptor()
    }}
    
    // API Repository
    single { HabitApiRepositoryImpl(get()) }
}

// Добавляем в платформенный модуль
```

**Файл:** `src/androidMain/kotlin/com/ecotrack/di/PlatformModule.kt`

```kotlin
package com.ecotrack.di

import org.koin.dsl.module
import com.ecotrack.data.local.databaseModule
import com.ecotrack.data.network.networkModule

actual val platformModule() = module {
    includes(databaseModule)
    includes(networkModule)
    
    // Sync Repository (объединяет БД и API)
    single { 
        com.ecotrack.data.repository.HabitSyncRepository(
            get(),  // Local repo
            get()   // API repo
        )
    }
}
```

---

## 4. Практика: Mock-сервер для тестирования

Для разработки без реального API создайте простой mock-эндпоинт:

**Варианты:**
1. **MockAPI (mockapi.io)** - бесплатный mock-сервер
2. **Firebase Firestore** - реальная БД с синхронизацией
3. **Supabase** - open-source альтернатива Firebase

### Пример Mock API (JSON):

```json
// GET /habits - список привычек
{
  "success": true,
  "data": [
    {
      "id": "1",
      "title": "Прогулка",
      "category": 0,
      "isCompleted": true,
      "createdAt": 1698234567000,
      "updatedAt": 1698234567000
    }
  ]
}

// POST /habits - создание привычки
{
  "success": true,
  "data": {
    "id": "2",
    "title": "Сортировка мусора",
    "category": 1,
    "isCompleted": false,
    "createdAt": 1698234567000,
    "updatedAt": 1698234567000
  }
}
```

---

## 📝 Домашнее задание (Модуль 5)

### Задание 1: Настройте Ktor Client
- Добавьте зависимости в `build.gradle.kts`
- Создайте Expect/Actual для HttpClient (Android/iOS)

### Задание 2: Создайте DTO и API Repository
- Определите `HabitDto` с аннотацией `@Serializable`
- Реализуйте CRUD методы в `HabitApiRepositoryImpl`

### Задание 3: Реализуйте Offline-First
- Создайте `HabitSyncRepository`, который сначала сохраняет в БД, затем синхронизирует с API
- Обработайте ошибки сети (не прерывайте выполнение)

### Задание 4: Добавьте интерцепторы
- Логирование всех запросов (для отладки)
- Retry для 5xx ошибок

### Задание 5: Тестирование синхронизации
- Отключите интернет (в эмуляторе)
- Добавьте привычку — должна сохраниться локально
- Включите интернет — проверьте, что синхронизация сработала

**Критерий сдачи:**
- Приложение работает без интернета (данные сохраняются локально)
- При наличии сети данные синхронизируются с API
- Ошибки сети обрабатываются корректно (не крашат приложение)

---

**💡 Совет:** Для тестирования используйте **Charles Proxy** или **mitmproxy** для перехвата и инспекции HTTP-запросов. Это поможет отладить проблемы с API.

**Важно:** Не забудьте добавить разрешения на интернет в `AndroidManifest.xml` и `Info.plist`:

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.INTERNET" />

<!-- Info.plist (iOS) -->
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

Удачи! В следующем модуле мы добавим красивые графики и анимации для визуализации данных.

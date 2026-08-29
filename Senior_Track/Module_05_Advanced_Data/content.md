# 📘 Модуль 3: Продвинутая работа с данными

**Добро пожаловать в третий модуль Senior Track!**  
В этом модуле мы выйдем за рамки базового SQLDelight и Ktor. Вы научитесь работать с GraphQL, реализуете real-time синхронизацию и создадите robust-систему кэширования с offline-first подходом.

**Цели модуля:**
1. Освоить GraphQL в KMP (Apollo Kotlin)
2. Реализовать WebSocket-синхронизацию для real-time данных
3. Создать multi-layer кэширование (memory + disk + network)
4. Настроить conflict resolution для офлайн-редактирования
5. Интегрировать несколько источников данных с fallback'ами

**Время выполнения:** ~30–40 часов (6 недель).

---

## 1. Введение: Ограничения REST API

В базовом курсе мы использовали REST API с Ktor Client. Но что делать, когда:
- Клиенту нужно 3 запроса для получения данных экрана? (N+1 проблема)
- Сервер возвращает лишние данные, которые не нужны на mobile?
- Нужно подписываться на изменения данных в real-time?
- Требуется сложная фильтрация и пагинация с разных источников?

**Решение:** GraphQL + WebSocket subscriptions.

---

## 2. Теория: GraphQL в KMP с Apollo Kotlin

### Что такое GraphQL?
GraphQL — это query language для API, где клиент сам определяет, какие данные ему нужны.

### Сравнение REST vs GraphQL:

**REST (традиционный подход):**
```http
GET /api/users/123         # Получаем пользователя
GET /api/users/123/skills  # Получаем навыки (дополнительный запрос)
GET /api/users/123/sessions # Получаем сессии (еще один запрос)
```

**GraphQL (один запрос):**
```graphql
query GetUserWithDetails {
  user(id: "123") {
    id
    name
    email
    skills {
      name
      level
    }
    sessions(limit: 10) {
      date
      status
      mentor {
        name
      }
    }
  }
}
```

### Преимущества GraphQL:
- ✅ Один запрос вместо N+1
- ✅ Клиент контролирует объем данных
- ✅ Типобезопасность (генерируется из схемы)
- ✅ Real-time subscriptions

---

## 3. Практика: Настройка Apollo Kotlin для SkillSync

### Шаг 1: Добавление зависимостей

В `shared/build.gradle.kts`:

```kotlin
plugins {
    // ... существующие плагины
    id("com.apollographql.apollo3") version "3.8.0"
}

kotlin {
    // ... существующие таргеты
    
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("com.apollographql.apollo3:apollo-runtime:3.8.0")
            }
        }
    }
}

// Настройка Apollo
apollo {
    service("skillSync") {
        packageName.set("com.skillsync.graphql")
        schemaFile.set(file("src/commonMain/graphql/schema.graphql"))
    }
}
```

### Шаг 2: Создание GraphQL схемы

Создайте `shared/src/commonMain/graphql/schema.graphql`:

```graphql
type Query {
  # Получить пользователя по ID
  user(id: ID!): User
  
  # Поиск навыков с фильтрацией
  skills(
    search: String, 
    category: SkillCategory,
    limit: Int = 20,
    offset: Int = 0
  ): SkillsConnection!
  
  # Получить доступные категории
  skillCategories: [SkillCategory!]!
}

type Mutation {
  # Создать навык пользователя
  createSkill(input: CreateSkillInput!): Skill!
  
  # Запланировать сессию
  scheduleSession(input: ScheduleSessionInput!): Session!
  
  # Отправить сообщение в чат
  sendMessage(input: SendMessageInput!): Message!
}

type Subscription {
  # Подписка на новые сообщения
  messageReceived(chatId: ID!): Message!
  
  # Подписка на изменения сессии
  sessionUpdated(sessionId: ID!): Session!
  
  # Подписка на онлайн-статус пользователя
  userOnlineStatus(userId: ID!): UserStatus!
}

type User {
  id: ID!
  name: String!
  email: String!
  avatarUrl: String
  skills: [Skill!]!
  sessions: [Session!]!
}

type Skill {
  id: ID!
  name: String!
  description: String
  category: SkillCategory!
  level: SkillLevel!
  rating: Float
}

enum SkillCategory {
  IT
  MUSIC
  SPORTS
  LANGUAGES
  ART
  OTHER
}

enum SkillLevel {
  BEGINNER
  INTERMEDIATE
  EXPERT
}

type Session {
  id: ID!
  mentorId: ID!
  menteeId: ID!
  skillId: ID!
  date: DateTime!
  status: SessionStatus!
}

enum SessionStatus {
  SCHEDULED
  COMPLETED
  CANCELLED
}

type Message {
  id: ID!
  chatId: ID!
  senderId: ID!
  content: String!
  timestamp: DateTime!
}

type SkillsConnection {
  edges: [SkillEdge!]!
  pageInfo: PageInfo!
}

type SkillEdge {
  node: Skill!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}

input CreateSkillInput {
  name: String!
  description: String
  category: SkillCategory!
}

input ScheduleSessionInput {
  mentorId: ID!
  skillId: ID!
  date: DateTime!
}

input SendMessageInput {
  chatId: ID!
  content: String!
}

type UserStatus {
  userId: ID!
  isOnline: Boolean!
  lastSeen: DateTime
}

scalar DateTime
```

### Шаг 3: Создание GraphQL запросов

Создайте `shared/src/commonMain/graphql/queries/getUser.graphql`:

```graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    name
    email
    avatarUrl
    skills {
      id
      name
      category
      level
    }
  }
}
```

Создайте `shared/src/commonMain/graphql/queries/searchSkills.graphql`:

```graphql
query SearchSkills($search: String, $category: SkillCategory, $limit: Int, $offset: Int) {
  skills(search: $search, category: $category, limit: $limit, offset: $offset) {
    edges {
      node {
        id
        name
        description
        category
        level
        rating
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

Создайте `shared/src/commonMain/graphql/subscriptions/messageReceived.graphql`:

```graphql
subscription MessageReceived($chatId: ID!) {
  messageReceived(chatId: $chatId) {
    id
    senderId
    content
    timestamp
  }
}
```

### Шаг 4: Использование Apollo Client в коде

Создайте `shared/src/commonMain/kotlin/data/remote/GqlClient.kt`:

```kotlin
package data.remote

import com.apollographql.apollo3.ApolloClient
import com.apollographql.apollo3.api.OperationOutput
import com.apollographql.apollo3.cache.normalized.ApolloCache
import com.apollographql.apollo3.cache.normalized.CacheHeaders
import com.apollographql.apollo3.cache.normalized.CacheKey
import com.apollographql.apollo3.cache.normalized.api.FieldPolicy
import com.apollographql.apollo3.cache.normalized.internal.toCacheKey
import com.apollographql.apollo3.exception.ApolloException
import com.apollographql.apollo3.network.ws.WebSocketNetworkTransport
import kotlinx.coroutines.flow.Flow

// Генерируемые классы из GraphQL запросов
import com.skillsync.graphql.*

class GqlClient(
    private val apolloClient: ApolloClient,
    private val wsTransport: WebSocketNetworkTransport? = null
) {
    // Выполнение query
    suspend fun getUser(id: String): Result<User> = result {
        apolloClient
            .query(GetUserQuery(id))
            .execute()
            .dataOrThrow()
            .user!!
    }
    
    // Выполнение query с кэшированием
    suspend fun getUserFromCache(id: String): Result<User> = result {
        apolloClient
            .query(GetUserQuery(id))
            .fetchPolicy(FetchPolicy.CacheAndNetwork)
            .execute()
            .dataOrThrow()
            .user!!
    }
    
    // Выполнение mutation
    suspend fun createSkill(input: CreateSkillInput): Result<Skill> = result {
        apolloClient
            .mutation(CreateSkillMutation(input))
            .execute()
            .dataOrThrow()
            .createSkill!!
    }
    
    // Подписка на subscription (real-time)
    fun subscribeToMessages(chatId: String): Flow<Message> {
        return apolloClient
            .subscription(MessageReceivedSubscription(chatId))
            .toFlow()
            .map { response ->
                when (response) {
                    is com.apollographql.apollo3.ApolloResponse.Data -> 
                        response.data.messageReceived!!
                    is com.apollographql.apollo3.ApolloResponse.Exception -> 
                        throw response.exception
                    else -> throw IllegalStateException("Unexpected response")
                }
            }
    }
}

// Extension функция для обработки ошибок
inline fun <T> result(block: () -> T): Result<T, ApolloException> =
    try {
        Result.Success(block())
    } catch (e: ApolloException) {
        Result.Failure(e)
    }
```

---

## 4. Теория: Multi-layer кэширование

### Стратегии кэширования:

#### 📦 Layer 1: Memory Cache (LruCache)
- Быстрый доступ (nanoseconds)
- Ограниченный размер
- Теряется при перезапуске приложения

#### 💾 Layer 2: Disk Cache (SQLDelight)
- Персистентное хранение
- Миллисекунды доступа
- Выживает после перезапуска

#### 🌐 Layer 3: Network (GraphQL/REST)
- Свежие данные с сервера
- Секунды доступа
- Требует интернет

### Offline-first подход:

```kotlin
class DataRepository(
    private val memoryCache: LruCache<String, Data>,
    private val diskCache: SqlDelightDatabase,
    private val networkClient: GqlClient
) {
    suspend fun getData(id: String): Result<Data> = result {
        // 1. Пытаемся получить из memory cache
        val fromMemory = memoryCache.get(id)
        if (fromMemory != null && !isExpired(fromMemory)) {
            return@result fromMemory
        }
        
        // 2. Пытаемся получить из disk cache
        val fromDisk = diskCache.getData(id)
        if (fromDisk != null && !isExpired(fromDisk)) {
            // Обновляем memory cache
            memoryCache.put(id, fromDisk)
            
            // 3. Асинхронно обновляем с сети (background)
            refreshFromNetwork(id)
            
            return@result fromDisk
        }
        
        // 4. Получаем с сети
        val fromNetwork = networkClient.getData(id)
        
        // Сохраняем в оба кэша
        diskCache.saveData(fromNetwork)
        memoryCache.put(id, fromNetwork)
        
        return@result fromNetwork
    }
    
    private suspend fun refreshFromNetwork(id: String) {
        try {
            val fresh = networkClient.getData(id)
            diskCache.saveData(fresh)
            memoryCache.put(id, fresh)
        } catch (e: Exception) {
            // Игнорируем ошибки в background обновлении
        }
    }
}
```

---

## 5. Практика: Реализация conflict resolution для офлайн-редактирования

### Проблема
Пользователь отредактировал данные офлайн. Другой пользователь тоже изменил те же данные на своем устройстве. Кто прав?

### Стратегии Conflict Resolution:

#### 1️⃣ Last Write Wins (LWW)
Последняя запись побеждает. Просто, но может потерять данные.

#### 2️⃣ Manual Merge
Пользователь сам решает конфликт. Сложно, но безопасно.

#### 3️⃣ Vector Clocks
Математически корректное разрешение конфликтов.

### Реализация LWW в SkillSync:

Создайте `shared/src/commonMain/kotlin/data/local/ConflictResolver.kt`:

```kotlin
package data.local

import kotlinx.datetime.Instant

data class VersionedData<T>(
    val data: T,
    val version: Long, // Timestamp или incrementing counter
    val updatedAt: Instant
)

class ConflictResolver<T> {
    /**
     * Last Write Wins стратегия
     */
    fun resolve(local: VersionedData<T>, remote: VersionedData<T>): VersionedData<T> {
        return if (remote.updatedAt > local.updatedAt) {
            remote
        } else {
            local
        }
    }
    
    /**
     * Merge стратегия для сложных объектов
     */
    fun merge(local: VersionedData<User>, remote: VersionedData<User>): VersionedData<User> {
        // Берем данные с более новой версией, но сохраняем локальные изменения
        val mergedData = if (remote.updatedAt > local.updatedAt) {
            remote.data.copy(
                // Сохраняем локальные поля, которые пользователь изменил
                localChanges = local.data.localChanges + remote.data.localChanges
            )
        } else {
            local.data
        }
        
        return VersionedData(
            data = mergedData,
            version = maxOf(local.version, remote.version),
            updatedAt = maxOf(local.updatedAt, remote.updatedAt)
        )
    }
}

// Extension для сравнения Instant
fun maxOf(a: Instant, b: Instant): Instant = if (a > b) a else b
```

---

## 6. Практика: Real-time синхронизация с WebSocket

### Шаг 1: Настройка WebSocket клиента

Создайте `shared/src/commonMain/kotlin/data/remote/WebSocketManager.kt`:

```kotlin
package data.remote

import io.ktor.client.*
import io.ktor.client.plugins.websocket.*
import io.ktor.websockets.*
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.receiveAsFlow

class WebSocketManager(
    private val client: HttpClient,
    private val url: String
) {
    private var webSocketSession: WebSocketSession? = null
    private val messageChannel = Channel<String>(Channel.BUFFERED)
    
    // Flow для получения сообщений
    val incomingMessages: Flow<String> = messageChannel.receiveAsFlow()
    
    suspend fun connect() {
        webSocketSession = client.webSocketSession(url)
        
        // Обработка входящих сообщений
        webSocketSession?.incoming?.forEach { frame ->
            when (frame) {
                is WebSocketFrame.Text -> {
                    messageChannel.trySend(frame.readText())
                }
                is WebSocketFrame.Close -> {
                    // Соединение закрыто, reconnect
                    reconnect()
                }
            }
        }
    }
    
    suspend fun send(message: String) {
        webSocketSession?.send(WebSocketFrame.Text(message))
    }
    
    suspend fun disconnect() {
        webSocketSession?.close()
        webSocketSession = null
    }
    
    private suspend fun reconnect() {
        // Реализация reconnection с exponential backoff
        delay(1000)
        connect()
    }
}
```

### Шаг 2: Интеграция с чатом

Создайте `shared/src/commonMain/kotlin/data/remote/ChatService.kt`:

```kotlin
package data.remote

import kotlinx.coroutines.flow.*

class ChatService(
    private val wsManager: WebSocketManager,
    private val gson: Gson // Для сериализации/десериализации
) {
    // Flow для сообщений конкретного чата
    fun getMessages(chatId: String): Flow<ChatMessage> {
        return wsManager.incomingMessages
            .filter { it.startsWith("message:$chatId:") }
            .map { jsonStr ->
                val data = jsonStr.removePrefix("message:$chatId:")
                gson.fromJson(data, ChatMessage::class.java)
            }
    }
    
    suspend fun sendMessage(chatId: String, content: String) {
        val message = ChatMessage(
            chatId = chatId,
            content = content,
            timestamp = Instant.now()
        )
        
        val json = gson.toJson(message)
        wsManager.send("send:$chatId:$json")
    }
}

data class ChatMessage(
    val id: String,
    val chatId: String,
    val senderId: String,
    val content: String,
    val timestamp: Instant
)
```

---

## 7. Практика: Интеграция нескольких источников данных с fallback'ами

### Паттерн: Repository с fallback'ами

Создайте `shared/src/commonMain/kotlin/data/CompositeRepository.kt`:

```kotlin
package data

sealed class DataSource {
    object GraphQL : DataSource()
    object REST : DataSource()
    object LocalDB : DataSource()
}

class CompositeRepository<T>(
    private val sources: List<Pair<DataSource, suspend () -> Result<T>>>
) {
    /**
     * Пытается получить данные из источников по порядку
     */
    suspend fun getData(): Result<T> {
        for ((source, fetcher) in sources) {
            try {
                val result = fetcher()
                if (result is Result.Success) {
                    // Успех - возвращаем данные с метаданными источника
                    return Result.Success(result.data)
                }
            } catch (e: Exception) {
                // Ошибка - пробуем следующий источник
                continue
            }
        }
        
        // Все источники недоступны
        return Result.Failure(DataUnavailableException())
    }
    
    /**
     * Получает данные с приоритетом: GraphQL > REST > LocalDB
     */
    suspend fun getDataWithPriority(): Result<T> {
        return getData()
    }
}

class DataUnavailableException : Exception("All data sources are unavailable")
```

### Использование в SkillSync:

```kotlin
// В модуле feature-skills
class SkillsRepository(
    private val gqlClient: GqlClient,
    private val restClient: RestApiClient,
    private val localDb: SqlDelightDatabase
) {
    private val compositeRepo = CompositeRepository<SkillList>(
        listOf(
            DataSource.GraphQL to { gqlClient.getSkills() },
            DataSource.REST to { restClient.getSkills() },
            DataSource.LocalDB to { localDb.getAllSkills() }
        )
    )
    
    suspend fun getSkills(): Result<SkillList> {
        return compositeRepo.getDataWithPriority()
    }
}
```

---

## 📝 Домашнее задание (Модуль 3)

### Задача: Создать robust-систему данных для SkillSync

**Требования:**

1. **GraphQL интеграция:**
   - Настройте Apollo Kotlin в проекте
   - Создайте минимум 3 query, 2 mutation, 1 subscription
   - Сгенерируйте типобезопасный код из схемы

2. **Multi-layer кэширование:**
   - Реализуйте 3 слоя: Memory (LruCache) → Disk (SQLDelight) → Network
   - Настройте TTL для каждого слоя (memory: 5min, disk: 1hour)
   - Реализуйте background refresh для устаревших данных

3. **Conflict resolution:**
   - Добавьте version tracking к сущностям (timestamp или counter)
   - Реализуйте LWW стратегию для простых объектов
   - Добавьте UI-индикатор "Данные могут быть неактуальны"

4. **Real-time синхронизация:**
   - Настройте WebSocket подключение для чата
   - Реализуйте reconnection с exponential backoff (1s, 2s, 4s, 8s...)
   - Добавьте heartbeat для поддержания соединения

5. **Offline-first подход:**
   - Приложение должно работать полностью офлайн
   - Очереди на отправку данных при отсутствии сети
   - Синхронизация при восстановлении соединения

**Критерии сдачи:**
- ✅ GraphQL запросы работают и возвращают данные
- ✅ Кэширование работает корректно (проверьте через Network Tab)
- ✅ Conflict resolution не теряет данные при синхронизации
- ✅ WebSocket reconnect работает после разрыва соединения
- ✅ Приложение полностью функционально в офлайн режиме

**Бонусные задания:**
- Реализуйте optimistic updates (мгновенное обновление UI до подтверждения сервером)
- Добавьте delta sync (синхронизация только изменений)
- Создайте кастомный cache policy для разных типов данных

---

## 💡 Советы по выполнению

1. **Начните с GraphQL:** Это основной источник данных, остальное - оптимизации.
2. **Тестируйте офлайн режим:** Отключите интернет и убедитесь, что приложение работает.
3. **Используйте Apollo DevTools:** Для отладки GraphQL запросов и кэша.
4. **Логгируйте все операции:** Понимание потока данных критически важно.
5. **Не усложняйте conflict resolution:** LWW достаточно для 90% случаев.

---

**Следующий модуль:** В Module_04_Native_Integration мы изучим C-interop, Objective-C++ биндинги и интеграцию нативных библиотек.

Удачи! 🚀

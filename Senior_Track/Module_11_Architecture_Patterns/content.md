# 📘 Модуль 7: Архитектурные паттерны

**Добро пожаловать в седьмой модуль Senior Track!**  
В этом модуле вы изучите современные архитектурные паттерны для KMP приложений: Clean Architecture, MVI (Model-View-Intent), Modularization. Научитесь создавать масштабируемую, тестируемую и поддерживаемую архитектуру.

**Цели модуля:**
1. Понять принципы Clean Architecture и применить их в KMP
2. Освоить MVI паттерн для управления состоянием UI
3. Научиться разделять приложение на модули (Feature Modules)
4. Реализовать Dependency Injection с Koin для KMP
5. Создать масштабируемую архитектуру для большого приложения

**Время выполнения:** ~35–45 часов (7 недель).

---

## 1. Введение: Эволюция архитектуры

### Проблемы монолитной архитектуры:

```
ПЛОХО (Монолит):
┌─────────────────────────┐
│      Main Activity      │
│  ┌───────────────────┐  │
│  │   All logic here  │  │ ← Всё смешано!
│  │ - UI             │  │
│  │ - Business Logic │  │
│  │ - Data Access    │  │
│  │ - Network        │  │
│  └───────────────────┘  │
└─────────────────────────┘

Проблемы:
❌ Сложно тестировать
❌ Сложно поддерживать
❌ Сложно масштабировать
❌ Высокая связанность (high coupling)
```

### Решение: Clean Architecture

```
ХОРОШО (Clean Architecture):
┌─────────────────────────┐
│      Presentation       │ ← UI (Compose/SwiftUI)
├─────────────────────────┤
│        Domain           │ ← Business Logic (100% KMP)
├─────────────────────────┤
│         Data            │ ← Data Sources (KMP + Native)
├─────────────────────────┤
│      Platform Specific  │ ← Android/iOS implementations
└─────────────────────────┘

Преимущества:
✅ Легко тестировать (изолированные слои)
✅ Легко поддерживать (четкие границы)
✅ Легко масштабировать (модульность)
✅ Низкая связанность (зависимости только вниз)
```

---

## 2. Теория: Clean Architecture в KMP

### Слои архитектуры:

#### 📱 Presentation Layer (UI)
**Ответственность:** Отображение данных и обработка пользовательских взаимодействий.

**Что здесь:**
- Compose/SwiftUI экраны
- ViewModels (MVI)
- UI компоненты

**Зависимости:** Только от Domain layer (через интерфейсы)

#### 🧠 Domain Layer (Business Logic)
**Ответственность:** Бизнес-логика приложения.

**Что здесь:**
- Use Cases (Interactors)
- Domain Entities
- Repository Interfaces

**Зависимости:** НЕ имеет зависимостей от других слоев! (чистый KMP)

#### 💾 Data Layer
**Ответственность:** Получение и сохранение данных.

**Что здесь:**
- Repository Implementations
- Local Data Sources (Database)
- Remote Data Sources (API)
- Mappers (Domain ↔ Data models)

**Зависимости:** От Domain layer (реализует интерфейсы)

#### 📲 Platform Layer
**Ответственность:** Платформенная специфика.

**Что здесь:**
- Android: Activities, Fragments, Native APIs
- iOS: ViewControllers, Swift UI, Native APIs

**Зависимости:** От всех слоев (точка входа)

---

### Принципы Clean Architecture:

#### 1️⃣ Dependency Rule
Зависимости направлены только ВНУТРЬ (к Domain).

```
Presentation → Domain ← Data
                      ↓
                Platform
```

#### 2️⃣ Separation of Concerns
Каждый слой отвечает только за свою задачу.

#### 3️⃣ Interface Segregation
Используйте интерфейсы для связи слоев, не реализации.

---

## 3. Практика: Рефакторинг к Clean Architecture

### Шаг 1: Структура проекта

```
shared/
├── src/
│   ├── commonMain/
│   │   ├── kotlin/
│   │   │   └── com/skillsync/
│   │   │       ├── domain/           ← Domain Layer (100% KMP)
│   │   │       │   ├── models/      (Entities)
│   │   │       │   ├── repository/  (Interfaces)
│   │   │       │   └── usecases/    (Business Logic)
│   │   │       ├── data/            ← Data Layer (KMP + Expect/Actual)
│   │   │       │   ├── repository/  (Implementations)
│   │   │       │   ├── local/       (Database)
│   │   │       │   ├── remote/      (API)
│   │   │       │   └── mappers/     (Domain ↔ Data)
│   │   │       └── presentation/    ← Presentation Layer (KMP UI)
│   │   │           ├── viewmodels/  (MVI ViewModels)
│   │   │           └── screens/     (Compose UI)
│   │   └── resources/
│   ├── androidMain/                 ← Android Platform Layer
│   │   └── kotlin/
│   │       └── com/skillsync/
│   │           ├── di/              (Android DI)
│   │           └── data/            (Platform-specific implementations)
│   ├── iosMain/                     ← iOS Platform Layer
│   │   └── shared/
│   │       ├── DI.swift             (iOS DI)
│   │       └── Data/                (Platform-specific implementations)
│   ├── commonTest/                  ← Tests для Domain
│   ├── androidUnitTest/             ← Android-specific tests
│   └── iosTest/                     ← iOS-specific tests
├── build.gradle.kts
└── detekt-config.yml

apps/
├── skillsync-android/               ← Android App Module
│   └── app/src/main/
│       ├── kotlin/ (MainActivity, DI setup)
│       └── res/
└── skillsync-ios/                   ← iOS App Module
    └── SkillSync/
        ├── AppDelegate.swift
        └── DI setup
```

### Шаг 2: Domain Layer - Entities и Repository Interfaces

Создайте `shared/src/commonMain/kotlin/com/skillsync/domain/models/Skill.kt`:

```kotlin
package com.skillsync.domain.models

// Domain Entity - чистая бизнес-сущность
data class Skill(
    val id: String,
    val name: String,
    val description: String?,
    val category: SkillCategory = SkillCategory.General,
    val difficulty: DifficultyLevel = DifficultyLevel.Beginner,
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis()
)

enum class SkillCategory {
    Programming,
    Design,
    Management,
    Marketing,
    General
}

enum class DifficultyLevel {
    Beginner,
    Intermediate,
    Advanced,
    Expert
}

// Domain Repository Interface - контракт для Data Layer
interface SkillsRepository {
    suspend fun getSkills(): Result<List<Skill>>
    suspend fun getSkillById(id: String): Result<Skill?>
    suspend fun createSkill(skill: Skill): Result<Skill>
    suspend fun updateSkill(skill: Skill): Result<Unit>
    suspend fun deleteSkill(id: String): Result<Unit>
    suspend fun searchSkills(query: String): Result<List<Skill>>
    suspend fun getSkillsByCategory(category: SkillCategory): Result<List<Skill>>
}

// Domain Use Case Interface
interface GetSkillsUseCase {
    suspend operator fun invoke(): Result<List<Skill>>
}

interface CreateSkillUseCase {
    suspend operator fun invoke(name: String, description: String?, category: SkillCategory): Result<Skill>
}

interface SearchSkillsUseCase {
    suspend operator fun invoke(query: String): Result<List<Skill>>
}
```

### Шаг 3: Domain Layer - Use Cases реализации

Создайте `shared/src/commonMain/kotlin/com/skillsync/domain/usecases/GetSkillsUseCase.kt`:

```kotlin
package com.skillsync.domain.usecases

import com.skillsync.domain.models.Skill
import com.skillsync.domain.repository.SkillsRepository

// Реализация Use Case - чистая бизнес-логика
class GetSkillsUseCase(
    private val repository: SkillsRepository
) : GetSkillsUseCase {
    
    override suspend fun invoke(): Result<List<Skill>> {
        return try {
            val skills = repository.getSkills()
            
            // Бизнес-логика: сортировка по дате создания
            val sortedSkills = skills.sortByDescending { it.createdAt }
            
            Result.success(sortedSkills)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

class CreateSkillUseCase(
    private val repository: SkillsRepository
) : CreateSkillUseCase {
    
    override suspend fun invoke(
        name: String, 
        description: String?, 
        category: com.skillsync.domain.models.SkillCategory
    ): Result<com.skillsync.domain.models.Skill> {
        
        // Валидация бизнес-правил
        if (name.isBlank()) {
            return Result.failure(IllegalArgumentException("Name cannot be empty"))
        }
        
        if (name.length > 100) {
            return Result.failure(IllegalArgumentException("Name too long"))
        }
        
        // Создание сущности
        val skill = com.skillsync.domain.models.Skill(
            id = generateId(),
            name = name.trim(),
            description = description?.trim(),
            category = category,
            difficulty = com.skillsync.domain.models.DifficultyLevel.Beginner,
            createdAt = System.currentTimeMillis(),
            updatedAt = System.currentTimeMillis()
        )
        
        return repository.createSkill(skill)
    }
    
    private fun generateId(): String {
        return System.currentTimeMillis().toString() + "-" + (1..8)
            .map { ('a'..'z')[(Math.random() * 26).toInt()] }
            .joinToString("")
    }
}

class SearchSkillsUseCase(
    private val repository: SkillsRepository
) : SearchSkillsUseCase {
    
    override suspend fun invoke(query: String): Result<List<com.skillsync.domain.models.Skill>> {
        if (query.isBlank()) {
            return Result.success(emptyList())
        }
        
        return repository.searchSkills(query)
    }
}
```

### Шаг 4: Data Layer - Repository Implementation

Создайте `shared/src/commonMain/kotlin/com/skillsync/data/repository/SkillsRepositoryImpl.kt`:

```kotlin
package com.skillsync.data.repository

import com.skillsync.domain.models.Skill
import com.skillsync.domain.repository.SkillsRepository
import com.skillsync.data.local.SkillsLocalDataSource
import com.skillsync.data.remote.SkillsRemoteDataSource

// Реализация Repository - стратегия cache-then-network
class SkillsRepositoryImpl(
    private val localDataSource: SkillsLocalDataSource,
    private val remoteDataSource: SkillsRemoteDataSource
) : SkillsRepository {
    
    override suspend fun getSkills(): Result<List<Skill>> {
        return try {
            // Сначала пробуем получить из кэша
            val cachedSkills = localDataSource.getAllSkills()
            
            if (cachedSkills.isNotEmpty()) {
                // Если кэш не пустой, обновляем с сервера в background
                refreshSkillsInBackground()
                return Result.success(cachedSkills)
            } else {
                // Если кэш пустой, загружаем с сервера
                val remoteSkills = remoteDataSource.getSkills()
                localDataSource.saveAll(remoteSkills)
                Result.success(remoteSkills)
            }
        } catch (e: Exception) {
            // Если ошибка сети, возвращаем кэш или ошибку
            val cachedSkills = localDataSource.getAllSkills()
            if (cachedSkills.isNotEmpty()) {
                Result.success(cachedSkills)
            } else {
                Result.failure(e)
            }
        }
    }
    
    override suspend fun getSkillById(id: String): Result<Skill?> {
        return try {
            val skill = localDataSource.getSkillById(id)
            Result.success(skill)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override suspend fun createSkill(skill: Skill): Result<Skill> {
        return try {
            // Сначала сохраняем локально для оптимистичного UI
            localDataSource.saveSkill(skill)
            
            // Затем синхронизируем с сервером
            val created = remoteDataSource.createSkill(skill)
            
            // Обновляем локальную копию с данными от сервера
            localDataSource.updateSkill(created)
            
            Result.success(created)
        } catch (e: Exception) {
            // Если ошибка сети, удаляем локальную запись
            localDataSource.deleteSkill(skill.id)
            Result.failure(e)
        }
    }
    
    override suspend fun updateSkill(skill: Skill): Result<Unit> {
        return try {
            val updated = remoteDataSource.updateSkill(skill)
            localDataSource.updateSkill(updated)
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override suspend fun deleteSkill(id: String): Result<Unit> {
        return try {
            remoteDataSource.deleteSkill(id)
            localDataSource.deleteSkill(id)
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override suspend fun searchSkills(query: String): Result<List<Skill>> {
        return try {
            val results = remoteDataSource.searchSkills(query)
            
            // Кэшируем результаты поиска (опционально)
            // localDataSource.cacheSearchResults(query, results)
            
            Result.success(results)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override suspend fun getSkillsByCategory(category: com.skillsync.domain.models.SkillCategory): Result<List<Skill>> {
        return try {
            val results = remoteDataSource.getSkillsByCategory(category)
            Result.success(results)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    // Background refresh - не блокирует основной поток
    private suspend fun refreshSkillsInBackground() {
        // Реализация будет в отдельном coroutine scope
    }
}
```

### Шаг 5: Presentation Layer - MVI ViewModel

Создайте `shared/src/commonMain/kotlin/com/skillsync/presentation/viewmodels/SkillsViewModel.kt`:

```kotlin
package com.skillsync.presentation.viewmodels

import com.skillsync.domain.models.Skill
import com.skillsync.domain.usecases.GetSkillsUseCase
import com.skillsync.domain.usecases.CreateSkillUseCase
import com.skillsync.domain.usecases.SearchSkillsUseCase
import kotlinx.coroutines.flow.*

// MVI Pattern: Model-View-Intent

// 1. Intent - действия пользователя
sealed class SkillsIntent {
    object LoadSkills : SkillsIntent()
    data class SearchSkills(val query: String) : SkillsIntent()
    data class CreateSkill(
        val name: String,
        val description: String?,
        val category: com.skillsync.domain.models.SkillCategory
    ) : SkillsIntent()
    
    data class UpdateSkill(val skill: Skill) : SkillsIntent()
    data class DeleteSkill(val id: String) : SkillsIntent()
}

// 2. State - состояние UI
data class SkillsUiState(
    val skills: List<Skill> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val searchQuery: String = "",
    val isSearching: Boolean = false
) {
    val isEmpty = skills.isEmpty() && !isLoading && error == null
    val hasError = error != null
}

// 3. ViewModel - обрабатывает Intents и обновляет State
class SkillsViewModel(
    private val getSkillsUseCase: GetSkillsUseCase,
    private val createSkillUseCase: CreateSkillUseCase,
    private val searchSkillsUseCase: SearchSkillsUseCase
) : ViewModel {
    
    // StateFlow для реактивного обновления UI
    private val _state = MutableStateFlow(SkillsUiState())
    val state: StateFlow<SkillsUiState> = _state.asStateFlow()
    
    // Обработчик Intent'ов
    fun onIntent(intent: SkillsIntent) {
        when (intent) {
            is SkillsIntent.LoadSkills -> loadSkills()
            is SkillsIntent.SearchSkills -> searchSkills(intent.query)
            is SkillsIntent.CreateSkill -> createSkill(
                intent.name, 
                intent.description, 
                intent.category
            )
            is SkillsIntent.UpdateSkill -> updateSkill(intent.skill)
            is SkillsIntent.DeleteSkill -> deleteSkill(intent.id)
        }
    }
    
    private suspend fun loadSkills() {
        _state.update { it.copy(isLoading = true, error = null) }
        
        try {
            val result = getSkillsUseCase.invoke()
            
            _state.update { currentState ->
                when (result) {
                    is Result.Success -> {
                        currentState.copy(
                            skills = result.data,
                            isLoading = false
                        )
                    }
                    is Result.Failure -> {
                        currentState.copy(
                            isLoading = false,
                            error = result.exception.message ?: "Unknown error"
                        )
                    }
                }
            }
        } catch (e: Exception) {
            _state.update { 
                it.copy(
                    isLoading = false,
                    error = e.message ?: "Unknown error"
                )
            }
        }
    }
    
    private suspend fun searchSkills(query: String) {
        _state.update { it.copy(searchQuery = query, isSearching = true) }
        
        try {
            val result = searchSkillsUseCase.invoke(query)
            
            _state.update { currentState ->
                when (result) {
                    is Result.Success -> {
                        currentState.copy(
                            skills = result.data,
                            isSearching = false
                        )
                    }
                    is Result.Failure -> {
                        currentState.copy(
                            isSearching = false,
                            error = result.exception.message
                        )
                    }
                }
            }
        } catch (e: Exception) {
            _state.update { 
                it.copy(
                    isSearching = false,
                    error = e.message
                )
            }
        }
    }
    
    private suspend fun createSkill(name: String, description: String?, category: com.skillsync.domain.models.SkillCategory) {
        try {
            val result = createSkillUseCase.invoke(name, description, category)
            
            _state.update { currentState ->
                when (result) {
                    is Result.Success -> {
                        currentState.copy(
                            skills = listOf(result.data) + currentState.skills,
                            error = null
                        )
                    }
                    is Result.Failure -> {
                        currentState.copy(
                            error = result.exception.message
                        )
                    }
                }
            }
        } catch (e: Exception) {
            _state.update { 
                it.copy(error = e.message)
            }
        }
    }
    
    private suspend fun updateSkill(skill: Skill) {
        // Реализация обновления...
    }
    
    private suspend fun deleteSkill(id: String) {
        _state.update { 
            it.copy(
                skills = it.skills.filter { s -> s.id != id }
            )
        }
    }
}

// Base ViewModel с lifecycle management
abstract class ViewModel {
    // Здесь будет управление жизненным циклом (dispose на destroy)
}

// Extension для Result type
sealed class Result<out T> {
    data class Success<out T>(val data: T) : Result<T>()
    data class Failure(val exception: Exception) : Result<Nothing>()
    
    inline fun <R> map(transform: (T) -> R): Result<R> = when (this) {
        is Success -> Success(transform(data))
        is Failure -> this as Result<R>
    }
}

fun <T> Result.success(data: T): Result<T> = Result.Success(data)
fun Result.failure(exception: Exception): Result<Nothing> = Result.Failure(exception)
```

---

## 4. Практика: Dependency Injection с Koin для KMP

### Шаг 1: Настройка Koin

Создайте `shared/build.gradle.kts` (добавьте зависимости):

```kotlin
dependencies {
    // Koin для DI
    implementation("io.insert-koin:koin-core:3.5.0")
    implementation("io.insert-koin:koin-core-coroutines:3.5.0")
    
    // Koin для Android (в androidMain)
    // implementation("io.insert-koin:koin-android:3.5.0")
}
```

### Шаг 2: Модули DI для KMP

Создайте `shared/src/commonMain/kotlin/com/skillsync/di/DomainModule.kt`:

```kotlin
package com.skillsync.di

import com.skillsync.domain.usecases.*
import org.koin.dsl.module

// Domain Module - Use Cases (не зависят от платформы)
val domainModule = module {
    // Use Cases
    single<GetSkillsUseCase> { 
        GetSkillsUseCase(get<SkillsRepository>())
    }
    
    single<CreateSkillUseCase> { 
        CreateSkillUseCase(get<SkillsRepository>())
    }
    
    single<SearchSkillsUseCase> { 
        SearchSkillsUseCase(get<SkillsRepository>())
    }
}

// Data Module - Repository Implementation (KMP + Expect/Actual)
val dataModule = module {
    // Repository implementation
    single<SkillsRepository> { 
        SkillsRepositoryImpl(
            localDataSource = get(),
            remoteDataSource = get()
        )
    }
    
    // Local Data Source (будет реализовано через expect/actual)
    single { SkillsLocalDataSourceImpl(getDatabase()) }
    
    // Remote Data Source (будет реализовано через expect/actual)
    single { SkillsRemoteDataSourceImpl(getHttpClient()) }
}

// Presentation Module - ViewModels (KMP UI)
val presentationModule = module {
    single { 
        SkillsViewModel(
            getSkillsUseCase = get(),
            createSkillUseCase = get(),
            searchSkillsUseCase = get()
        )
    }
}

// Combined Module для commonMain
val sharedModule = module {
    includes(
        domainModule,
        dataModule,
        presentationModule
    )
}

// Expect/Actual для платформенных зависимостей
expect fun getDatabase(): Any // Будет реализовано в androidMain/iosMain
expect fun getHttpClient(): Any // Будет реализовано в androidMain/iosMain
```

Создайте `shared/src/androidMain/kotlin/com/skillsync/di/PlatformModule.android.kt`:

```kotlin
package com.skillsync.di

import org.koin.dsl.module
import org.koin.android.ext.koin.androidContext

// Android-specific DI module
val androidPlatformModule = module {
    // Android-specific implementations
    single { AndroidDatabaseProvider.getDatabase(androidContext()) }
    single { AndroidHttpClientProvider.getHttpClient() }
    
    // Android-specific services (Notification, Biometric, etc.)
    single { AndroidNotificationManager(androidContext()) }
    single { AndroidBiometricAuth(androidContext()) }
}

actual fun getDatabase() = TODO("Use Koin injection in Android")
actual fun getHttpClient() = TODO("Use Koin injection in Android")

// Main DI setup для Android
fun createAndroidModules() = listOf(
    sharedModule,
    androidPlatformModule
)
```

Создайте `shared/src/iosMain/kotlin/com/skillsync/di/PlatformModule.ios.kt`:

```kotlin
package com.skillsync.di

import org.koin.dsl.module

// iOS-specific DI module  
val iosPlatformModule = module {
    // iOS-specific implementations
    single { IOSDatabaseProvider.getDatabase() }
    single { IOSHttpClientProvider.getHttpClient() }
    
    // iOS-specific services
    single { IOSNotificationManager() }
    single { IOSBiometricAuth() }
}

actual fun getDatabase() = TODO("Use Koin injection in iOS")
actual fun getHttpClient() = TODO("Use Koin injection in iOS")

// Main DI setup для iOS
fun createIOSModules() = listOf(
    sharedModule,
    iosPlatformModule
)
```

---

## 5. Практика: Modularization (Feature Modules)

### Проблема монолитного shared модуля:
- Медленная сборка при изменении любого кода
- Сложно разделять ответственность между командами
- Все зависимости загружаются даже если не нужны

### Решение: Feature Modules

```
shared/                    ← Core functionality (общее)
├── core-common/          ← Общие утилиты, extensions
├── core-data/            ← Data layer (Database, Network)
├── core-domain/          ← Domain models, Use Cases interfaces
└── core-presentation/    ← Common UI components

features/                  ← Feature modules (каждый - отдельный Gradle module)
├── feature-skill-sync/   ← Основной фича: синхронизация навыков
│   ├── src/commonMain/
│   │   └── kotlin/com/skillsync/featureskillssync/
│   │       ├── domain/     (Feature-specific Use Cases)
│   │       ├── data/       (Feature-specific Repository impls)
│   │       └── presentation/(Feature-specific UI + ViewModels)
│   └── build.gradle.kts  (зависит от core-*)
├── feature-ar-card/      ← Фича: AR визитки
├── feature-profile/      ← Фича: профиль пользователя
└── feature-settings/     ← Фича: настройки

apps/                     ← App modules (собирают features)
├── skillsync-android/    ← Android app (зависит от features/*)
└── skillsync-ios/        ← iOS app (зависит от features/*)
```

### Настройка Feature Module

Создайте `features/feature-skill-sync/build.gradle.kts`:

```kotlin
plugins {
    kotlin("multiplatform")
    id("com.android.library")
}

kotlin {
    androidTarget()
    
    iosX64()
    iosArm64()
    iosSimulatorArm64()
    
    sourceSets {
        val commonMain by getting {
            dependencies {
                // Зависимости от core модулей
                implementation(project(":shared:core-common"))
                implementation(project(":shared:core-domain"))
                implementation(project(":shared:core-data"))
                implementation(project(":shared:core-presentation"))
                
                // Koin для DI
                implementation("io.insert-koin:koin-core:3.5.0")
            }
        }
        
        val androidMain by getting {
            dependencies {
                implementation("io.insert-koin:koin-android:3.5.0")
            }
        }
    }
}

// Feature-specific DI module
android {
    namespace = "com.skillsync.feature.skillssync"
}
```

Создайте `features/feature-skill-sync/src/commonMain/kotlin/com/skillsync/featureskillssync/di/FeatureModule.kt`:

```kotlin
package com.skillsync.featureskillssync.di

import org.koin.dsl.module
import com.skillsync.featureskillssync.domain.usecases.*
import com.skillsync.featureskillssync.data.repository.*
import com.skillsync.featureskillssync.presentation.viewmodels.*

// Feature-specific DI module
val skillSyncFeatureModule = module {
    // Feature Use Cases
    single<GetSkillsUseCase> { 
        GetSkillsUseCase(get<SkillsRepository>())
    }
    
    // Feature Repository Implementation
    single<SkillsRepository> { 
        SkillsRepositoryImpl(get(), get())
    }
    
    // Feature ViewModels
    single { 
        SkillsViewModel(get(), get(), get())
    }
}
```

---

## 📝 Домашнее задание (Модуль 7)

### Задача: Рефакторинг SkillSync к Clean Architecture с Modularization

**Требования:**

1. **Clean Architecture:**
   - Разделите код на 3 слоя: Domain, Data, Presentation
   - Domain слой должен быть 100% KMP без платформенных зависимостей
   - Используйте интерфейсы для связи слоев

2. **MVI Pattern:**
   - Реализуйте MVI ViewModels для всех экранов
   - Используйте StateFlow для реактивного UI
   - Обработайте все состояния: Loading, Success, Error

3. **Dependency Injection:**
   - Настройте Koin для DI в KMP
   - Создайте модули: domainModule, dataModule, presentationModule
   - Реализуйте expect/actual для платформенных зависимостей

4. **Modularization:**
   - Разделите shared на core-* модули (core-common, core-domain, etc.)
   - Создайте feature-skill-sync как отдельный Gradle module
   - Настройте зависимости между модулями

5. **Тестирование:**
   - Напишите unit-тесты для Use Cases (Domain слой)
   - Напишите unit-тесты для ViewModels (Presentation слой)
   - Достижение: 80%+ coverage

**Критерии сдачи:**
- ✅ Четкое разделение на слои (Domain, Data, Presentation)
- ✅ Domain слой не зависит от других слоев
- ✅ MVI ViewModels с StateFlow для всех экранов
- ✅ Koin настроен и работает на обеих платформах
- ✅ Feature module вынесен в отдельный Gradle модуль
- ✅ 80%+ coverage для Domain и Presentation слоев

**Бонусные задания:**
- Создайте еще 2 feature модуля (feature-profile, feature-settings)
- Реализуйте offline-first архитектуру с background sync
- Добавьте support для multiple databases (SQLite + Realm)

---

## 💡 Советы по выполнению

1. **Начните с Domain слоя:** Это ядро архитектуры - сделайте его идеальным.
2. **Используйте интерфейсы:** Все связи между слоями через интерфейсы из Domain.
3. **Не усложняйте prematurely:** Начните с одного feature module, потом масштабируйте.
4. **Тестируйте каждый слой отдельно:** Domain - unit тесты, Data - integration тесты.
5. **Документируйте архитектуру:** Создайте ARCHITECTURE.md с диаграммами и объяснениями.

---

**Следующий модуль:** В Module_08_Advanced_Topics мы изучим продвинутые темы: C++ interop, машинное обучение в KMP, advanced concurrency patterns.

Удачи! 🚀

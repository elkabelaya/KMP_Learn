# 📘 Модуль 3: Архитектура и Dependency Injection

**Добро пожаловать в третий модуль!**  
Теперь, когда у нас есть работающий UI, пришло время превратить «спагетти-код» в профессиональную архитектуру. Мы научимся разделять UI от бизнес-логики, внедрять зависимости и писать тестируемый код.

**Цели модуля:**
1. Понять паттерны архитектуры (MVVM vs MVI) и выбрать подходящий для KMP
2. Освоить Dependency Injection с помощью **Koin** (стандарт для KMP)
3. Реализовать Clean Architecture: UI, Domain, Data слои
4. Настроить Expect/Actual для платформенных зависимостей

**Время выполнения:** ~10–12 часов.

---

## 1. Теория: Почему нужна архитектура?

### Проблема «Спагетти-кода»
Без архитектуры код выглядит так:

```kotlin
// ❌ ПЛОХО: Логика внутри UI
@Composable
fun HomeScreen() {
    var habits by remember { mutableStateOf(listOf()) }
    
    Button(onClick = { 
        // Тут и валидация, и запрос к БД, и сетевой вызов...
        val newHabit = saveToDatabase(title) 
        syncWithServer(newHabit)
        habits += newHabit
    }) { Text("Добавить") }
}
```

**Проблемы:**
- Нельзя протестировать логику отдельно от UI
- Код дублируется при изменении требований
- Сложно поддерживать и масштабировать

### Решение: Clean Architecture
Мы разделим приложение на 3 слоя:

```
┌─────────────────────────────┐
│         UI Layer            │ ← Compose Screens, ViewModels
├─────────────────────────────┤
│       Domain Layer          │ ← UseCases, Интерфейсы (100% KMP)
├─────────────────────────────┤
│        Data Layer          │ ← Repository, БД, API (Expect/Actual)
└─────────────────────────────┘
```

**Принципы:**
1. **UI** знает только о **Domain** (через ViewModel)
2. **Domain** не зависит от платформ (только интерфейсы)
3. **Data** реализует интерфейсы из Domain для каждой платформы

---

## 2. Теория: Паттерны архитектуры

### MVVM (Model-View-ViewModel) — Рекомендуемый для KMP
**Как работает:**
- **View (UI):** Compose-экраны, отображают данные из ViewModel
- **ViewModel:** Хранит состояние (StateFlow), содержит бизнес-логику
- **Model (Repository):** Интерфейс для доступа к данным

**Преимущества:**
- Простота понимания
- Отличная поддержка в Compose (StateFlow, CollectAsState)
- Легкое тестирование

### MVI (Model-View-Intent) — Альтернатива
**Как работает:**
- **View:** Отправляет Intent (события) в ViewModel
- **ViewModel:** Обрабатывает Intent, возвращает State
- **State:** Иммутабельный объект со всем состоянием

**Когда использовать:** Сложные приложения с множеством состояний. Для EcoTrack достаточно MVVM.

---

## 3. Теория: Dependency Injection (Koin)

**Dependency Injection (DI)** — это паттерн, который позволяет внедрять зависимости извне вместо создания их внутри класса.

**Почему Koin?**
- Легковесный (нет кодогенерации)
- Полная поддержка KMP
- Простой синтаксис

### Базовая структура Koin:

```kotlin
// Определение модуля (что и как создавать)
val appModule = module {
    single { HabitRepository(...) }
    viewModel { HabitViewModel(get()) }
}

// Использование в коде
class HabitViewModel(
    private val repository: HabitRepository // Внедряется автоматически
) { ... }
```

### Expect/Actual для DI
В KMP нам нужно создать разные модули для разных платформ:

```kotlin
// commonMain
expect val platformModule(): Module

// androidMain
actual val platformModule() = module { ... }

// iosMain  
actual val platformModule() = module { ... }
```

---

## 4. Практика: Реализация Clean Architecture

### Шаг 1: Создаем структуру папок

```
src/commonMain/kotlin/com/ecotrack/
├── ui/                    # UI Layer (Compose)
│   ├── screens/          # Экраны
│   └── viewmodels/       # ViewModels
├── domain/                # Domain Layer (Бизнес-логика)
│   ├── model/            # Entity (независимые от БД)
│   └── usecase/          # UseCases (бизнес-правила)
├── data/                  # Data Layer (Реализации)
│   ├── repository/       # Repository реализации
│   └── local/            # БД (будет в модуле 4)
└── di/                    # Dependency Injection
    └── AppModule.kt      # Koin модули
```

### Шаг 2: Создаем Domain слой (Интерфейсы)

**Файл:** `commonMain/kotlin/com/ecotrack/domain/model/HabitEntity.kt`

```kotlin
package com.ecotrack.domain.model

// Entity — это чистая бизнес-модель, не зависит от БД
data class HabitEntity(
    val id: String = UUID.randomUUID().toString(),
    val title: String,
    val category: HabitCategory,
    val isCompleted: Boolean = false,
    val createdAt: Long = System.currentTimeMillis()
)

enum class HabitCategory {
    TRANSPORT,  // Транспорт (прогулка, велосипед)
    WASTE,      // Сортировка мусора
    WATER,      // Экономия воды
    PLASTIC     // Отказ от пластика
}
```

**Файл:** `commonMain/kotlin/com/ecotrack/domain/repository/HabitRepository.kt`

```kotlin
package com.ecotrack.domain.repository

import com.ecotrack.domain.model.HabitEntity
import kotlinx.coroutines.flow.Flow

// Интерфейс — контракт для всех платформ
interface HabitRepository {
    fun getHabits(): Flow<List<HabitEntity>>
    suspend fun addHabit(habit: HabitEntity): Result<Unit>
    suspend fun updateHabit(habit: HabitEntity): Result<Unit>
    suspend fun deleteHabit(id: String): Result<Unit>
    suspend fun getHabitsByCategory(category: HabitCategory): Flow<List<HabitEntity>>
}
```

### Шаг 3: Создаем UseCases (Бизнес-логика)

**Файл:** `commonMain/kotlin/com/ecotrack/domain/usecase/AddHabitUseCase.kt`

```kotlin
package com.ecotrack.domain.usecase

import com.ecotrack.domain.model.HabitEntity
import com.ecotrack.domain.repository.HabitRepository

// UseCase — один бизнес-сценарий
class AddHabitUseCase(
    private val repository: HabitRepository
) {
    suspend operator fun invoke(
        title: String,
        category: HabitCategory
    ): Result<HabitEntity> {
        
        // Валидация (бизнес-правила)
        if (title.isBlank()) {
            return Result.failure(Exception("Название не может быть пустым"))
        }
        
        if (title.length > 100) {
            return Result.failure(Exception("Название слишком длинное"))
        }
        
        // Создание сущности
        val habit = HabitEntity(
            title = title.trim(),
            category = category
        )
        
        // Сохранение через репозиторий
        return repository.addHabit(habit).map { habit }
    }
}
```

### Шаг 4: Создаем ViewModel (UI Layer)

**Файл:** `commonMain/kotlin/com/ecotrack/ui/viewmodels/HabitViewModel.kt`

```kotlin
package com.ecotrack.ui.viewmodels

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.ecotrack.domain.model.HabitEntity
import com.ecotrack.domain.usecase.AddHabitUseCase
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

// Состояние экрана
data class HabitUiState(
    val habits: List<HabitEntity> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

class HabitViewModel(
    private val addHabitUseCase: AddHabitUseCase,
    // В будущем добавим другие UseCases
) : ViewModel() {
    
    // StateFlow для состояния экрана
    private val _uiState = MutableStateFlow(HabitUiState())
    val uiState: StateFlow<HabitUiState> = _uiState.asStateFlow()
    
    // Запуск при инициализации
    init {
        loadHabits()
    }
    
    private fun loadHabits() {
        // Здесь будет подписка на репозиторий (в модуле 4)
        _uiState.value = _uiState.value.copy(
            isLoading = true,
            habits = listOf() // Заглушка
        )
    }
    
    fun addHabit(title: String, category: HabitCategory) {
        viewModelScope.launch {
            _uiState.value = _uiState.value.copy(isLoading = true, error = null)
            
            when (val result = addHabitUseCase(title, category)) {
                is Result.Success -> {
                    // Обновить список (в модуле 4)
                    _uiState.value = _uiState.value.copy(isLoading = false)
                }
                is Result.Failure -> {
                    _uiState.value = _uiState.value.copy(
                        isLoading = false,
                        error = result.exception.message
                    )
                }
            }
        }
    }
}
```

### Шаг 5: Настраиваем Koin (DI)

**Файл:** `commonMain/kotlin/com/ecotrack/di/AppModule.kt`

```kotlin
package com.ecotrack.di

import org.koin.core.module.Module
import org.koin.dsl.module

// Общий модуль (независимый от платформы)
val commonModule = module {
    // UseCases
    factory { AddHabitUseCase(get()) }
    
    // ViewModels
    viewModel { HabitViewModel(get()) }
}

// Ожидаем платформенный модуль
expect val platformModule(): Module

// Собираем все вместе
val appModules = listOf(
    commonModule,
    platformModule()
)
```

**Файл:** `androidMain/kotlin/com/ecotrack/di/PlatformModule.kt`

```kotlin
package com.ecotrack.di

import org.koin.core.module.Module
import org.koin.dsl.module

// Android-специфичные зависимости (БД, API)
actual val platformModule(): Module = module {
    // Здесь будет репозиторий (в модуле 4)
    single { /* HabitRepositoryImpl(...) */ }
}
```

**Файл:** `iosMain/kotlin/com/ecotrack/di/PlatformModule.kt`

```kotlin
package com.ecotrack.di

import org.koin.core.module.Module
import org.koin.dsl.module

actual val platformModule(): Module = module {
    // iOS-специфичные зависимости (CoreData, URLSession)
    single { /* HabitRepositoryImpl(...) */ }
}
```

### Шаг 6: Инициализация Koin в приложении

**Файл:** `androidMain/kotlin/com/ecotrack/MainActivity.kt`

```kotlin
package com.ecotrack

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.runtime.CompositionLocalProvider
import org.koin.android.ext.android.get
import org.koin.core.context.startKoin
import com.ecotrack.di.appModules

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Запускаем Koin
        startKoin {
            modules(appModules)
        }
        
        setContent {
            // Получаем ViewModel через Koin
            val viewModel: HabitViewModel = get()
            
            AppContent(viewModel = viewModel)
        }
    }
}
```

---

## 5. Практика: Обновляем UI с ViewModel

**Файл:** `commonMain/kotlin/com/ecotrack/ui/screens/HomeScreen.kt`

```kotlin
package com.ecotrack.ui.screens

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import com.ecotrack.domain.model.HabitCategory
import com.ecotrack.ui.viewmodels.HabitViewModel

@Composable
fun HomeScreen(
    viewModel: HabitViewModel, // Внедряется через Koin
    onNavigateToAdd: () -> Unit
) {
    // Наблюдаем за состоянием ViewModel
    val uiState by viewModel.uiState.collectAsState()
    
    var showAddDialog by remember { mutableStateOf(false) }
    var habitTitle by remember { mutableStateOf("") }
    
    Scaffold(
        floatingActionButton = {
            FloatingActionButton(onClick = onNavigateToAdd) {
                Icon(
                    imageVector = androidx.compose.material.icons.Icons.Default.Add,
                    contentDescription = "Добавить"
                )
            }
        }
    ) { padding ->
        Column(modifier = Modifier.fillMaxSize().padding(padding)) {
            // Заголовок
            Text(
                text = "Мои привычки",
                style = MaterialTheme.typography.headlineMedium,
                modifier = Modifier.padding(16.dp)
            )
            
            // Ошибка
            uiState.error?.let { error ->
                Card(
                    colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.errorContainer),
                    modifier = Modifier.padding(16.dp)
                ) {
                    Text(
                        text = error,
                        modifier = Modifier.padding(16.dp),
                        color = MaterialTheme.colorScheme.onErrorContainer
                    )
                }
            }
            
            // Список привычек (LazyColumn из модуля 2)
            LazyColumn {
                items(uiState.habits, key = { it.id }) { habit ->
                    HabitCard(
                        habit = habit,
                        onClick = { /* Открыть детали */ }
                    )
                }
            }
            
            // Диалог добавления (упрощенно)
            if (showAddDialog) {
                AlertDialog(
                    onDismissRequest = { showAddDialog = false },
                    title = { Text("Добавить привычку") },
                    text = {
                        OutlinedTextField(
                            value = habitTitle,
                            onValueChange = { habitTitle = it },
                            label = { Text("Название") }
                        )
                    },
                    confirmButton = {
                        Button(
                            onClick = {
                                viewModel.addHabit(habitTitle, HabitCategory.WASTE)
                                showAddDialog = false
                            }
                        ) {
                            Text("Сохранить")
                        }
                    },
                    dismissButton = {
                        TextButton(onClick = { showAddDialog = false }) {
                            Text("Отмена")
                        }
                    }
                )
            }
        }
    }
}
```

---

## 📝 Домашнее задание (Модуль 3)

### Задание 1: Реализуйте Clean Architecture
- Создайте папки `domain/`, `data/`, `ui/viewmodels/` в `commonMain`
- Перенесите код из предыдущих модулей в соответствующие слои
- Убедитесь, что UI не знает о БД или сети

### Задание 2: Настройте Koin
- Установите зависимость `koin-core` и `koin-android` в `build.gradle.kts`
- Создайте модули DI для UseCases и ViewModels
- Инициализируйте Koin в `MainActivity` (Android) и `MainViewController` (iOS)

### Задание 3: Создайте UseCases
- `AddHabitUseCase` — добавление с валидацией (не пустое, не >100 символов)
- `GetHabitsUseCase` — получение списка (пока возвращает заглушку)
- `DeleteHabitUseCase` — удаление по ID

### Задание 4: Обновите ViewModel
- Добавьте обработку ошибок (показывайте Snackbar при ошибке)
- Реализуйте загрузку данных (isLoading состояние)

### Задание 5: Тестирование архитектуры
- Попробуйте удалить все зависимости из `HomeScreen` (кроме ViewModel)
- Убедитесь, что код компилируется — это значит архитектура правильная

**Критерий сдачи:**
- Код разделен на 3 слоя (UI, Domain, Data)
- Koin настроен и внедряет зависимости
- UseCases содержат бизнес-логику (валидация)
- ViewModel управляет состоянием через StateFlow

---

**💡 Совет:** Не пытайтесь сразу реализовать репозиторий с БД. В этом модуле создайте заглушку (FakeRepository), которая возвращает тестовые данные. Реальную БД сделаем в модуле 4.

**Важно:** Если код не компилируется — проверьте, что все интерфейсы из Domain реализованы в Data слое. Это самая частая ошибка при внедрении архитектуры.

Удачи! В следующем модуле мы подключим реальную базу данных SQLDelight и сделаем персистентность.

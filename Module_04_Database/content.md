# 📘 Модуль 4: Работа с данными (Local Database)

**Добро пожаловать в четвертый модуль!**  
Теперь, когда у нас есть архитектура, пришло время сделать данные постоянными. Мы подключим **SQLDelight** — кроссплатформенную базу данных, которая работает одинаково на Android и iOS.

**Цели модуля:**
1. Понять, как работает SQLDelight в KMP
2. Настроить схему БД и миграции
3. Реализовать CRUD операции (Create, Read, Update, Delete)
4. Подключить БД к репозиторию и ViewModel

**Время выполнения:** ~10–12 часов.

---

## 1. Теория: Выбор СУБД для KMP

### Варианты для KMP:

| Технология | Плюсы | Минусы | Рекомендация |
|------------|-------|--------|--------------|
| **SQLDelight** | Нативная БД, типобезопасность, миграции | Нужно писать SQL-запросы | ✅ **Рекомендуется** |
| Room Multiplatform | Знакомая API (как на Android) | Только SQLite, меньше фич | Для простых проектов |
| Realm | Быстрая, объектная модель | Проприетарная лицензия | Для enterprise |
| CoreData (iOS) + Room (Android) | Нативные API | Дублирование кода | ❌ Не рекомендуется |

### Почему SQLDelight?
- **Типобезопасность:** Компилятор проверяет SQL-запросы
- **Миграции:** Автоматическое обновление схемы БД
- **Нативная производительность:** SQLite на Android, SQLite/CoreData на iOS
- **KMP-first:** Создан специально для Kotlin Multiplatform

---

## 2. Теория: Как работает SQLDelight

### Архитектура SQLDelight в KMP

```
┌─────────────────────────────┐
│      commonMain             │ ← SQL-запросы (.sq)
│   └── com/ecotrack/db/     │    и драйверы (expect)
├─────────────────────────────┤
│      androidMain            │ ← SQLite драйвер (actual)
├─────────────────────────────┤
│      iosMain                │ ← SQLite драйвер (actual)
└─────────────────────────────┘
```

### Ключевые концепции:

1. **SQL-файлы (.sq):** Пишете SQL-запросы в отдельных файлах
2. **Компиляция:** Gradle компилирует SQL в Kotlin-код
3. **Driver:** Expect/Actual для разных платформ
4. **Appender:** Типобезопасная вставка параметров

---

## 3. Практика: Настройка SQLDelight

### Шаг 1: Добавляем зависимости

**Файл:** `shared/build.gradle.kts` (или `app/build.gradle.kts`)

```kotlin
dependencies {
    // SQLDelight для commonMain
    implementation("app.cash.sqldelight:runtime:2.0.2")
    
    // SQLDelight для Android
    androidImplementation("app.cash.sqldelight:sqlite-driver:2.0.2")
    
    // SQLDelight для iOS (нативный SQLite)
    iosX64Implementation("app.cash.sqldelight:native-driver:2.0.2")
    iosArm64Implementation("app.cash.sqldelight:native-driver:2.0.2")
    iosSimulatorArm64Implementation("app.cash.sqldelight:native-driver:2.0.2")
}

// Настройка SQLDelight
sqldelight {
    databases {
        create("EcoTrackDatabase") {
            packageName = "com.ecotrack.data.local.db"
        }
    }
}
```

### Шаг 2: Создаем схему БД

**Файл:** `src/commonMain/sq/com/ecotrack/db/Habit.sql`

```sql
-- Таблица привычек
CREATE TABLE habit (
    id TEXT PRIMARY KEY NOT NULL,
    title TEXT NOT NULL,
    category INTEGER NOT NULL,  -- 0=TRANSPORT, 1=WASTE, 2=WATER, 3=PLASTIC
    is_completed INTEGER NOT NULL DEFAULT 0,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);

-- Индекс для быстрого поиска по категории
CREATE INDEX habit_category_idx ON habit(category);

-- Query: Получить все привычки (отсортировано по дате)
type HabitQuery = 
    SELECT id, title, category, is_completed, created_at, updated_at
    FROM habit
    ORDER BY created_at DESC;

-- Query: Получить привычки по категории
type HabitByCategoryQuery = 
    SELECT id, title, category, is_completed, created_at, updated_at
    FROM habit
    WHERE category = ?
    ORDER BY created_at DESC;

-- Query: Получить одну привычку по ID
type HabitByIdQuery = 
    SELECT id, title, category, is_completed, created_at, updated_at
    FROM habit
    WHERE id = ?;

-- Query: Удалить привычку по ID
type DeleteHabitQuery = 
    DELETE FROM habit WHERE id = ?;

-- Query: Обновить привычку
type UpdateHabitQuery = 
    UPDATE habit 
    SET title = ?, category = ?, is_completed = ?, updated_at = ?
    WHERE id = ?;

-- Query: Вставить новую привычку
type InsertHabitQuery = 
    INSERT INTO habit (id, title, category, is_completed, created_at, updated_at)
    VALUES (?, ?, ?, ?, ?, ?);
```

### Шаг 3: Создаем Expect/Actual для драйвера

**Файл:** `src/commonMain/kotlin/com/ecotrack/data/local/DatabaseModule.kt`

```kotlin
package com.ecotrack.data.local

import app.cash.sqldelight.db.SqlDriver

// Ожидаем драйвер от платформы
expect fun createDatabaseDriver(): SqlDriver
```

**Файл:** `src/androidMain/kotlin/com/ecotrack/data/local/DatabaseModule.kt`

```kotlin
package com.ecotrack.data.local

import android.content.Context
import app.cash.sqldelight.db.SqlDriver
import app.cash.sqldelight.driver.android.AndroidSqliteDriver
import com.ecotrack.data.local.db.EcoTrackDatabase

actual fun createDatabaseDriver(): SqlDriver {
    // В реальном приложении передаем context из Application
    return AndroidSqliteDriver(EcoTrackDatabase.Schema, /* context */, "ecotrack.db")
}
```

**Файл:** `src/iosMain/kotlin/com/ecotrack/data/local/DatabaseModule.kt`

```kotlin
package com.ecotrack.data.local

import app.cash.sqldelight.db.SqlDriver
import app.cash.sqldelight.driver.native.NativeSqliteDriver
import com.ecotrack.data.local.db.EcoTrackDatabase

actual fun createDatabaseDriver(): SqlDriver {
    return NativeSqliteDriver(EcoTrackDatabase.Schema, "ecotrack.db")
}
```

### Шаг 4: Создаем Repository с БД

**Файл:** `src/commonMain/kotlin/com/ecotrack/data/local/HabitRepositoryImpl.kt`

```kotlin
package com.ecotrack.data.local

import app.cash.sqldelight.coroutines.asFlow
import app.cash.sqldelight.coroutines.mapToList
import com.ecotrack.data.local.db.Habit
import com.ecotrack.data.local.db.EcoTrackDatabase
import com.ecotrack.domain.model.HabitCategory
import com.ecotrack.domain.model.HabitEntity
import com.ecotrack.domain.repository.HabitRepository
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.withContext

// Реализация репозитория с SQLDelight
class HabitRepositoryImpl(
    private val database: EcoTrackDatabase
) : HabitRepository {
    
    override fun getHabits(): Flow<List<HabitEntity>> {
        return database.habitQueries
            .getAllHabits()
            .asFlow()
            .mapToList(Dispatchers.Default)
            .map { list -> 
                list.map { it.toDomainEntity() } 
            }
    }
    
    override suspend fun addHabit(habit: HabitEntity): Result<Unit> = withContext(Dispatchers.Default) {
        try {
            database.transaction {
                database.habitQueries.insertHabit(
                    id = habit.id,
                    title = habit.title,
                    category = habit.category.ordinal.toLong(),
                    isCompleted = if (habit.isCompleted) 1L else 0L,
                    createdAt = habit.createdAt,
                    updatedAt = System.currentTimeMillis()
                )
            }
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override suspend fun updateHabit(habit: HabitEntity): Result<Unit> = withContext(Dispatchers.Default) {
        try {
            database.transaction {
                database.habitQueries.updateHabit(
                    title = habit.title,
                    category = habit.category.ordinal.toLong(),
                    isCompleted = if (habit.isCompleted) 1L else 0L,
                    updatedAt = System.currentTimeMillis(),
                    id = habit.id
                )
            }
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override suspend fun deleteHabit(id: String): Result<Unit> = withContext(Dispatchers.Default) {
        try {
            database.habitQueries.deleteHabit(id = id)
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override fun getHabitsByCategory(category: HabitCategory): Flow<List<HabitEntity>> {
        return database.habitQueries
            .getHabitsByCategory(category.ordinal.toLong())
            .asFlow()
            .mapToList(Dispatchers.Default)
            .map { list -> 
                list.map { it.toDomainEntity() } 
            }
    }
}

// Extension function для маппинга из БД в Domain
private fun Habit.toDomainEntity(): HabitEntity {
    return HabitEntity(
        id = this.id,
        title = this.title,
        category = HabitCategory.entries[this.category.toInt()],
        isCompleted = this.is_completed == 1L,
        createdAt = this.created_at
    )
}
```

### Шаг 5: Создаем Database Module для Koin

**Файл:** `src/commonMain/kotlin/com/ecotrack/data/local/DatabaseModule.kt`

```kotlin
package com.ecotrack.data.local

import app.cash.sqldelight.db.SqlDriver
import com.ecotrack.data.local.db.EcoTrackDatabase
import com.ecotrack.domain.repository.HabitRepository
import org.koin.core.module.Module
import org.koin.dsl.module

// Модуль для БД (общий код)
val databaseModule = module {
    // Создаем драйвер через expect/actual
    single<SqlDriver> { createDatabaseDriver() }
    
    // Создаем базу данных
    single { EcoTrackDatabase(get()) }
    
    // Реализация репозитория
    single<HabitRepository> { HabitRepositoryImpl(get()) }
}

// Добавляем в платформенный модуль
expect val platformModule(): Module
```

**Файл:** `src/androidMain/kotlin/com/ecotrack/di/PlatformModule.kt`

```kotlin
package com.ecotrack.di

import org.koin.core.module.Module
import org.koin.dsl.module
import com.ecotrack.data.local.databaseModule

actual val platformModule(): Module = module {
    includes(databaseModule)
}
```

**Файл:** `src/iosMain/kotlin/com/ecotrack/di/PlatformModule.kt`

```kotlin
package com.ecotrack.di

import org.koin.core.module.Module
import org.koin.dsl.module
import com.ecotrack.data.local.databaseModule

actual val platformModule(): Module = module {
    includes(databaseModule)
}
```

### Шаг 6: Обновляем ViewModel для работы с БД

**Файл:** `src/commonMain/kotlin/com/ecotrack/ui/viewmodels/HabitViewModel.kt`

```kotlin
package com.ecotrack.ui.viewmodels

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.ecotrack.domain.model.HabitEntity
import com.ecotrack.domain.repository.HabitRepository
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

data class HabitUiState(
    val habits: List<HabitEntity> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

class HabitViewModel(
    private val repository: HabitRepository // Внедряется через Koin
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(HabitUiState())
    val uiState: StateFlow<HabitUiState> = _uiState.asStateFlow()
    
    init {
        loadHabits()
    }
    
    private fun loadHabits() {
        viewModelScope.launch {
            _uiState.value = _uiState.value.copy(isLoading = true)
            
            try {
                repository.getHabits()
                    .collect { habits ->
                        _uiState.value = _uiState.value.copy(
                            isLoading = false,
                            habits = habits
                        )
                    }
            } catch (e: Exception) {
                _uiState.value = _uiState.value.copy(
                    isLoading = false,
                    error = e.message ?: "Ошибка загрузки"
                )
            }
        }
    }
    
    fun addHabit(title: String, category: HabitCategory) {
        viewModelScope.launch {
            _uiState.value = _uiState.value.copy(isLoading = true, error = null)
            
            val habit = HabitEntity(
                title = title.trim(),
                category = category
            )
            
            when (val result = repository.addHabit(habit)) {
                is Result.Success -> {
                    // Перезагружаем список
                    loadHabits()
                }
                is Result.Failure -> {
                    _uiState.value = _uiState.value.copy(
                        isLoading = false,
                        error = result.exception.message ?: "Ошибка сохранения"
                    )
                }
            }
        }
    }
    
    fun deleteHabit(id: String) {
        viewModelScope.launch {
            when (val result = repository.deleteHabit(id)) {
                is Result.Success -> {
                    loadHabits()
                }
                is Result.Failure -> {
                    _uiState.value = _uiState.value.copy(
                        error = result.exception.message ?: "Ошибка удаления"
                    )
                }
            }
        }
    }
}
```

---

## 4. Практика: Миграции БД

### Что такое миграции?
Миграции — это SQL-скрипты для обновления схемы БД при изменении приложения.

**Файл:** `src/commonMain/sq/com/ecotrack/db/1_create_habit_table.sql`

```sql
-- Версия 1: Создаем таблицу
CREATE TABLE habit (
    id TEXT PRIMARY KEY NOT NULL,
    title TEXT NOT NULL,
    category INTEGER NOT NULL,
    is_completed INTEGER NOT NULL DEFAULT 0,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);

CREATE INDEX habit_category_idx ON habit(category);
```

**Файл:** `src/commonMain/sq/com/ecotrack/db/2_add_description.sql`

```sql
-- Версия 2: Добавляем поле description (если нужно в будущем)
ALTER TABLE habit ADD COLUMN description TEXT;
```

SQLDelight автоматически применяет миграции по порядку.

---

## 📝 Домашнее задание (Модуль 4)

### Задание 1: Настройте SQLDelight
- Добавьте зависимости в `build.gradle.kts`
- Создайте `.sql` файл с таблицей `habit`
- Сгенерируйте код (Gradle синхронизация)

### Задание 2: Реализуйте CRUD операции
- `insertHabit` — добавление новой привычки
- `getAllHabits` — получение всех (Flow)
- `updateHabit` — обновление (например, отметить выполненной)
- `deleteHabit` — удаление по ID

### Задание 3: Подключите БД к Koin
- Создайте `databaseModule` с драйвером и репозиторием
- Добавьте его в `platformModule()` для Android и iOS

### Задание 4: Обновите ViewModel
- Подпишитесь на `repository.getHabits()` через Flow
- Реализуйте добавление и удаление привычек

### Задание 5: Тестирование
- Запустите приложение на Android-эмуляторе
- Добавьте 3-5 привычек
- Перезапустите приложение — данные должны сохраниться

**Критерий сдачи:**
- БД создается автоматически при первом запуске
- Привычки сохраняются после перезапуска приложения
- Список обновляется в реальном времени (Flow)
- Ошибки обрабатываются корректно

---

**💡 Совет:** Если SQL-запросы не компилируются — проверьте синтаксис. SQLDelight очень строгий к типам данных. Используйте `?` для параметров, а не вставляйте значения напрямую.

**Важно:** Для iOS-симулятора может потребоваться перезапуск симулятора после изменения схемы БД.

Удачи! В следующем модуле мы подключим сетевое взаимодействие и сделаем синхронизацию данных между устройствами.

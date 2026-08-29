# 📘 Модуль 8: Тестирование, Оптимизация и Релиз

**Добро пожаловать в финальный модуль!**  
Приложение готово, но прежде чем публиковать его в сторах, нужно протестировать, оптимизировать и подготовить к релизу. Мы пройдем весь путь от unit-тестов до публикации в App Store и Google Play.

**Цели модуля:**
1. Написать unit-тесты для commonMain (ViewModel, Repository)
2. Провести UI-тестирование на Android и iOS
3. Оптимизировать производительность (размер APK, время запуска)
4. Подготовить приложение к релизу в сторах

**Время выполнения:** ~15–20 часов.

---

## 1. Теория: Тестирование в KMP

### Виды тестирования:

| Тип | Что тестирует | Где выполняется | Инструменты |
|-----|---------------|-----------------|-------------|
| **Unit** | Логику (ViewModel, Repository) | commonTest | JUnit 5, Kotest |
| **Integration** | Взаимодействие компонентов | androidTest / iosTest | Testcontainers, MockK |
| **UI** | Интерфейс и сценарии | Эмулятор/Симулятор | Compose Test, XCUITest |
| **E2E** | Полные сценарии пользователя | Реальные устройства | Maestro, Detox |

### Структура тестов в KMP:

```
shared/
├── src/
│   ├── commonMain/kotlin/          # Основной код
│   ├── commonTest/kotlin/          # Общие тесты (unit)
│   ├── androidUnitTest/kotlin/     # Android unit-тесты
│   ├── androidInstrumentedTest/kotlin/  # Android UI-тесты
│   └── iosTest/kotlin/            # iOS тесты (XCTest)
```

---

## 2. Практика: Unit-тесты для commonMain

### Шаг 1: Добавляем зависимости для тестов

**Файл:** `shared/build.gradle.kts`

```kotlin
dependencies {
    // JUnit 5 для commonTest
    commonTestImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
    commonTestImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
    
    // MockK для моков (только JVM)
    androidUnitTestImplementation("io.mockk:mockk:1.13.8")
    
    // Turbine для тестирования Flow (от JetBrains)
    commonTestImplementation("app.cash.turbine:turbine:1.0.0")
}

kotlin {
    sourceSets {
        commonTest.dependencies {
            implementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
            implementation("app.cash.turbine:turbine:1.0.0")
        }
        
        androidUnitTest.dependencies {
            implementation("io.mockk:mockk:1.13.8")
        }
    }
}
```

### Шаг 2: Пишем unit-тесты для ViewModel

**Файл:** `src/commonTest/kotlin/com/ecotrack/ui/viewmodels/HabitViewModelTest.kt`

```kotlin
package com.ecotrack.ui.viewmodels

import app.cash.turbine.test
import com.ecotrack.domain.model.HabitCategory
import com.ecotrack.domain.repository.HabitRepository
import kotlinx.coroutines.test.runTest
import kotlin.test.Test
import kotlin.test.assertEquals

// Mock репозитория (для commonTest используем простой mock)
class HabitRepositoryMock : HabitRepository {
    val habits = mutableListOf<com.ecotrack.domain.model.HabitEntity>()
    
    override fun getHabits() = kotlinx.coroutines.flow.flow {
        emit(habits.toList())
    }
    
    override suspend fun addHabit(habit: com.ecotrack.domain.model.HabitEntity): Result<Unit> {
        habits.add(habit.copy(id = "test-id-${habits.size + 1}"))
        return Result.success(Unit)
    }
    
    override suspend fun updateHabit(habit: com.ecotrack.domain.model.HabitEntity): Result<Unit> {
        val index = habits.indexOfFirst { it.id == habit.id }
        if (index != -1) {
            habits[index] = habit
        }
        return Result.success(Unit)
    }
    
    override suspend fun deleteHabit(id: String): Result<Unit> {
        habits.removeAll { it.id == id }
        return Result.success(Unit)
    }
    
    override fun getHabitsByCategory(category: com.ecotrack.domain.model.HabitCategory) = 
        kotlinx.coroutines.flow.flow {
            emit(habits.filter { it.category == category })
        }
}

class HabitViewModelTest {
    
    @Test
    fun `adding habit updates UI state`() = runTest {
        // Arrange
        val repository = HabitRepositoryMock()
        val viewModel = HabitViewModel(repository)
        
        // Act
        viewModel.addHabit("Прогулка", HabitCategory.TRANSPORT)
        
        // Assert с использованием Turbine для тестирования Flow
        viewModel.uiState.test {
            val state = awaitItem()
            assertEquals(1, state.habits.size)
            assertEquals("Прогулка", state.habits[0].title)
        }
    }
    
    @Test
    fun `deleting habit removes it from list`() = runTest {
        // Arrange
        val repository = HabitRepositoryMock()
        val viewModel = HabitViewModel(repository)
        
        // Добавляем привычку
        viewModel.addHabit("Прогулка", HabitCategory.TRANSPORT)
        
        // Ждем обновления состояния
        viewModel.uiState.test {
            val initialState = awaitItem()
            assertEquals(1, initialState.habits.size)
            
            // Удаляем привычку
            viewModel.deleteHabit(initialState.habits[0].id)
            
            // Проверяем, что список пуст
            val finalState = awaitItem()
            assertEquals(0, finalState.habits.size)
        }
    }
    
    @Test
    fun `error is shown when repository fails`() = runTest {
        // Arrange - создаем репозиторий, который всегда возвращает ошибку
        class FailingRepository : HabitRepository {
            override fun getHabits() = kotlinx.coroutines.flow.flow { 
                throw Exception("Database error")
            }
            
            override suspend fun addHabit(habit: com.ecotrack.domain.model.HabitEntity): Result<Unit> = 
                Result.failure(Exception("Database error"))
            
            override suspend fun updateHabit(habit: com.ecotrack.domain.model.HabitEntity): Result<Unit> = 
                Result.failure(Exception("Database error"))
            
            override suspend fun deleteHabit(id: String): Result<Unit> = 
                Result.failure(Exception("Database error"))
            
            override fun getHabitsByCategory(category: com.ecotrack.domain.model.HabitCategory) = 
                kotlinx.coroutines.flow.flow { }
        }
        
        val repository = FailingRepository()
        val viewModel = HabitViewModel(repository)
        
        // Act & Assert
        viewModel.uiState.test {
            val state = awaitItem()
            assertEquals(true, state.isLoading)
            
            // Ждем следующего состояния с ошибкой
            val errorState = awaitItem()
            assertEquals(false, errorState.isLoading)
            assertNotNull(errorState.error)
        }
    }
}
```

### Шаг 3: Тесты для Repository с MockK (Android)

**Файл:** `src/androidUnitTest/kotlin/com/ecotrack/data/local/HabitRepositoryImplTest.kt`

```kotlin
package com.ecotrack.data.local

import app.cash.sqldelight.db.SqlDriver
import app.cash.sqldelight.driver.jdbc.sqlite.JdbcSqliteDriver
import com.ecotrack.data.local.db.EcoTrackDatabase
import io.mockk.every
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import org.junit.Assert.assertEquals
import org.junit.Test

class HabitRepositoryImplTest {
    
    private val driver = JdbcSqliteDriver(JdbcConnectionConfig.inMemory)
    private val database = EcoTrackDatabase(driver)
    private val repository = HabitRepositoryImpl(database)
    
    @Test
    fun `adding habit saves to database`() = runTest {
        // Arrange
        val habit = com.ecotrack.domain.model.HabitEntity(
            title = "Прогулка",
            category = com.ecotrack.domain.model.HabitCategory.TRANSPORT
        )
        
        // Act
        val result = repository.addHabit(habit)
        
        // Assert
        assertEquals(Result.Success(Unit), result)
    }
    
    @Test
    fun `getting habits returns list from database`() = runTest {
        // Arrange - добавляем несколько привычек
        repository.addHabit(com.ecotrack.domain.model.HabitEntity(
            title = "Прогулка",
            category = com.ecotrack.domain.model.HabitCategory.TRANSPORT
        ))
        
        repository.addHabit(com.ecotrack.domain.model.HabitEntity(
            title = "Сортировка мусора",
            category = com.ecotrack.domain.model.HabitCategory.WASTE
        ))
        
        // Act & Assert
        repository.getHabits().test {
            val habits = awaitItem()
            assertEquals(2, habits.size)
        }
    }
}

// Конфигурация для in-memory БД (только для тестов)
data class JdbcConnectionConfig(
    val url: String = "jdbc:sqlite::memory:",
    val user: String? = null,
    val password: String? = null
) {
    companion object {
        val inMemory = JdbcConnectionConfig()
    }
}
```

---

## 3. Практика: UI-тесты с Compose Test

### Шаг 1: Добавляем зависимости для UI-тестов

**Файл:** `shared/build.gradle.kts`

```kotlin
dependencies {
    // Compose Test для Android
    androidInstrumentedTestImplementation("androidx.compose.ui:ui-test-junit4:1.5.4")
    androidInstrumentedTestImplementation("androidx.compose.ui:ui-test-manifest:1.5.4")
    
    // Для iOS используем XCTest (настраивается в Xcode)
}

kotlin {
    sourceSets {
        androidInstrumentedTest.dependencies {
            implementation("androidx.compose.ui:ui-test-junit4:1.5.4")
            implementation("androidx.compose.ui:ui-test-manifest:1.5.4")
        }
    }
}
```

### Шаг 2: Пишем UI-тесты для Android

**Файл:** `src/androidInstrumentedTest/kotlin/com/ecotrack/ui/screens/HomeScreenTest.kt`

```kotlin
package com.ecotrack.ui.screens

import androidx.compose.ui.test.*
import androidx.test.ext.junit.rules.ActivityScenarioRule
import androidx.test.platform.app.InstrumentationRegistry
import com.ecotrack.ui.theme.EcoTrackTheme
import org.junit.Rule
import org.junit.Test

class HomeScreenTest {
    
    @get:Rule
    val activityRule = ActivityScenarioRule(MainActivity())
    
    @Test
    fun testHabitListDisplaysCorrectly() {
        activityRule.scenario.onActivity { activity ->
            // Запускаем Compose тест
            val composeTestRule = createComposeRule()
            
            // Проверяем, что список привычек отображается
            composeTestRule.onNodeWithText("Мои привычки").assertIsDisplayed()
            
            // Проверяем, что кнопка добавления видна
            composeTestRule.onNodeWithContentDescription("Добавить привычку")
                .assertIsDisplayed()
        }
    }
    
    @Test
    fun testAddingHabitUpdatesList() {
        activityRule.scenario.onActivity { activity ->
            val composeTestRule = createComposeRule()
            
            // Нажимаем кнопку добавления
            composeTestRule.onNodeWithContentDescription("Добавить привычку")
                .performClick()
            
            // Вводим название привычки
            composeTestRule.onNodeWithText("Название")
                .performTextInput("Прогулка")
            
            // Выбираем категорию (если есть)
            composeTestRule.onNodeWithText("Транспорт")
                .performClick()
            
            // Нажимаем кнопку сохранения
            composeTestRule.onNodeWithText("Сохранить")
                .performClick()
            
            // Проверяем, что привычка появилась в списке
            composeTestRule.onNodeWithText("Прогулка")
                .assertIsDisplayed()
        }
    }
}

// Helper для создания Compose Test Rule
fun createComposeRule() = androidx.compose.ui.test.createAndroidComposeRule<MainActivity>()
```

---

## 4. Практика: Оптимизация производительности

### Шаг 1: Анализ размера APK/AAB

**Файл:** `shared/build.gradle.kts` (оптимизация)

```kotlin
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(
                getDefaultProguardFile('proguard-android-optimize.txt'),
                'proguard-rules.pro'
            )
        }
    }
}

kotlin {
    // Включаем оптимизацию для iOS
    targets.all {
        compilations.all {
            kotlinOptions {
                freeCompilerArgs += "-Xopt-in=kotlin.RequiresOptIn"
            }
        }
    }
}
```

**Файл:** `shared/proguard-rules.pro`

```proguard
# Сохраняем классы, которые нужны для рефлексии
-keep class com.ecotrack.** { *; }

# Сохраняем DTO для сериализации
-keep class com.ecotrack.data.network.dto.** { *; }

# Исключаем ненужные библиотеки
-dontwarn kotlinx.serialization.**
-dontwarn kotlin.Metadata

# Оптимизация SQLDelight
-keep class app.cash.sqldelight.** { *; }

# Оптимизация Ktor
-dontwarn io.ktor.**
```

### Шаг 2: Оптимизация времени запуска

**Файл:** `src/androidMain/kotlin/com/ecotrack/MainActivity.kt`

```kotlin
class MainActivity : ComponentActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Откладываем инициализацию тяжелых компонентов
        lifecycleScope.launch {
            delay(100) // Даем время для отрисовки первого кадра
            
            // Инициализируем Koin и другие тяжелые зависимости
            setupKoin()
            
            setContent {
                EcoTrackTheme {
                    // ... UI код
                }
            }
        }
    }
}
```

### Шаг 3: Профилирование с Android Profiler и Instruments

**Инструменты:**
- **Android:** Android Studio Profiler (CPU, Memory, Network)
- **iOS:** Xcode Instruments (Time Profiler, Allocations, Energy Log)

**Что проверять:**
1. **Memory leaks:** Утечки памяти в ViewModel и репозиториях
2. **CPU usage:** Тяжелые операции на main thread
3. **Network requests:** Избыточные запросы к API
4. **Battery usage:** Частые wakeups и background задачи

---

## 5. Практика: Подготовка к релизу

### Шаг 1: Настройка подписи для Android

**Файл:** `shared/build.gradle.kts`

```kotlin
android {
    signingConfigs {
        create("release") {
            storeFile = file("../keystore/ecotrack.keystore")
            storePassword = System.getenv("KEYSTORE_PASSWORD") ?: ""
            keyAlias = "ecotrack"
            keyPassword = System.getenv("KEY_PASSWORD") ?: ""
        }
    }
    
    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
            isMinifyEnabled = true
        }
    }
}
```

**Создание keystore:**
```bash
keytool -genkeypair \
  -v \
  -keystore ecotrack.keystore \
  -alias ecotrack \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000
```

### Шаг 2: Настройка подписи для iOS (Xcode)

1. **Apple Developer Account** ($99/год)
2. **Создать Certificate:** Certificates, Identifiers & Profiles → Certificates → +
3. **Создать Provisioning Profile:** App Store Distribution
4. **Настроить в Xcode:** Signing & Capabilities → Auto-signing

### Шаг 3: Публикация в Google Play Console

**Шаги:**
1. Создать приложение в [Google Play Console](https://play.google.com/console)
2. Заполнить информацию о приложении (описание, скриншоты, ключевые слова)
3. Загрузить AAB файл (не APK!)
4. Настроить контент-рейтинг и политику конфиденциальности
5. Отправить на ревью (обычно 3-7 дней)

**Требования:**
- **AAB формат** (не APK)
- **Target SDK 34+** (на момент написания)
- **Политика конфиденциальности** (обязательно)
- **Скриншоты** для всех размеров устройств

### Шаг 4: Публикация в App Store Connect

**Шаги:**
1. Создать приложение в [App Store Connect](https://appstoreconnect.apple.com)
2. Заполнить информацию (описание, ключевые слова, скриншоты)
3. Загрузить бинарник через Xcode или Transporter
4. Настроить App Privacy (обязательно с iOS 14+)
5. Отправить на ревью (обычно 2-5 дней)

**Требования:**
- **iOS 15+** как минимальная версия (рекомендуется)
- **App Privacy Nutrition Label** (обязательно)
- **Скриншоты** для всех размеров iPhone и iPad

---

## 6. Практика: CI/CD с GitHub Actions

**Файл:** `.github/workflows/build.yml`

```yaml
name: Build and Test

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build-android:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew
      
      - name: Build Android
        run: ./gradlew :shared:assembleRelease
      
      - name: Run tests
        run: ./gradlew :shared:testAndroidUnitTest
      
      - name: Upload APK
        uses: actions/upload-artifact@v3
        with:
          name: ecotrack-android-apk
          path: shared/build/outputs/apk/release/*.apk

  build-ios:
    runs-on: macos-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Build iOS
        run: ./gradlew :shared:assembleXCFramework
      
      - name: Upload XCFramework
        uses: actions/upload-artifact@v3
        with:
          name: ecotrack-ios-framework
          path: shared/build/compose/cinterop/ecotrack.xcframework
```

---

## 📝 Домашнее задание (Модуль 8)

### Задание 1: Напишите unit-тесты
- Тесты для всех ViewModel (минимум 3 теста на каждую)
- Тесты для Repository с реальной БД (SQLDelight + JdbcDriver)

### Задание 2: Напишите UI-тесты
- Тест для добавления привычки (Android)
- Тест для просмотра статистики

### Задание 3: Оптимизируйте приложение
- Включите ProGuard/R8 для Android
- Уменьшите размер APK до < 20MB
- Проверьте время запуска (< 3 секунд)

### Задание 4: Подготовьте к релизу
- Создайте keystore для Android
- Настройте подписи в Xcode (если есть Apple Developer аккаунт)
- Заполните политики конфиденциальности

### Задание 5: Настройте CI/CD
- Создайте GitHub Actions workflow для автоматического билда и тестов

**Критерий сдачи:**
- Все тесты проходят (100% coverage для commonMain)
- APK < 20MB, время запуска < 3с
- CI/CD pipeline работает автоматически

---

## 🎉 Поздравляю! Вы завершили курс!

**Что вы освоили:**
✅ Kotlin Multiplatform (KMP) архитектура  
✅ Compose Multiplatform для UI  
✅ SQLDelight для локальной БД  
✅ Ktor Client для сети  
✅ Expect/Actual паттерн  
✅ Push-уведомления и deep links  
✅ Unit-тесты и UI-тесты  
✅ Оптимизация и релиз в сторах  

**Ваш уровень:** Middle Kotlin Multiplatform Developer

### Что изучать дальше:
1. **KMP Server-Side:** Ktor + KMM для бэкенда
2. **Compose Desktop:** Приложения для Windows/macOS/Linux
3. **KMP Web:** Компиляция в JavaScript/WASM
4. **Архитектурные паттерны:** MVI, Clean Architecture в KMP
5. **Advanced Testing:** MockK, Testcontainers, UI Automation

### Полезные ресурсы:
- [Официальная документация KMP](https://kotlinlang.org/docs/multiplatform.html)
- [Compose Multiplatform Samples](https://github.com/JetBrains/compose-multiplatform-samples)
- [Kotlin Slack](https://kotlinlang.slack.com/) (каналы #multiplatform, #compose)
- [KotlinConf YouTube](https://www.youtube.com/c/KotlinConf)

**Удачи в разработке! 🚀**

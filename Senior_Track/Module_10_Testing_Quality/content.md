# 📘 Модуль 6: Тестирование и качество кода

**Добро пожаловать в шестой модуль Senior Track!**  
В этом модуле вы построите полную пирамиду тестирования для KMP приложения: от unit-тестов до UI-автоматизации. Научитесь писать тестируемый код, настраивать CI/CD и поддерживать высокое качество кода.

**Цели модуля:**
1. Написать unit-тесты для commonMain кода (JUnit, MockK)
2. Создать integration-тесты для платформенного кода (XCTest, JUnit)
3. Настроить UI-тесты для Android (Espresso) и iOS (XCUITest)
4. Настроить CI/CD пайплайны для автоматического тестирования
5. Внедрить code quality инструменты (Detekt, SwiftLint)

**Время выполнения:** ~30–40 часов (6 недель).

---

## 1. Введение: Пирамида тестирования

### Уровни тестирования:

```
         /\
        /  \
       / UI \          ← 10% (медленные, дорогие)
      /------\
     /        \
    /  E2E    \         ← 20% (средние)
   /----------\
  /            \
 / Integration  \        ← 30% (быстрые)
/----------------\
/                \
|     Unit       |        ← 40% (очень быстрые, дешевые)
------------------
```

### Типы тестов:

#### 🔬 Unit-тесты
- Тестируют отдельные функции/классы в изоляции
- Очень быстрые (< 10ms)
- Не требуют платформенного кода
- **Цель:** 70%+ coverage для commonMain

#### 🔗 Integration-тесты
- Тестируют взаимодействие компонентов (Repository + Database)
- Быстрые (< 1s)
- Могут требовать платформенный код
- **Цель:** Критичные пути работы

#### 🎮 UI-тесты (E2E)
- Тестируют пользовательские сценарии
- Медленные (> 5s)
- Хрупкие (ломаются при изменении UI)
- **Цель:** Критичные пользовательские сценарии

---

## 2. Практика: Unit-тесты для commonMain

### Шаг 1: Настройка тестового окружения

Создайте `shared/build.gradle.kts`:

```kotlin
plugins {
    kotlin("multiplatform")
}

kotlin {
    // ... существующие таргеты
    
    sourceSets {
        val commonTest by getting {
            dependencies {
                // JUnit для KMP
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
                
                // MockK для моков
                implementation("io.mockk:mockk:1.13.4")
                
                // Truth для ассертов (альтернатива AssertJ)
                implementation("com.google.truth:truth:1.1.5")
            }
        }
        
        val androidUnitTest by getting {
            dependencies {
                implementation("junit:junit:4.13.2")
            }
        }
        
        val iosX64Test by getting
        val iosArm64Test by getting
        val iosSimulatorArm64Test by getting
    }
}

tasks {
    // Настройка тестов для разных платформ
    withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompile> {
        kotlinOptions {
            freeCompilerArgs += "-Xopt-in=kotlin.RequiresOptIn"
        }
    }
}
```

### Шаг 2: Написание unit-тестов для Repository

Создайте `shared/src/commonTest/kotlin/data/SkillsRepositoryTest.kt`:

```kotlin
package data

import io.mockk.*
import kotlinx.coroutines.test.runTest
import kotlin.test.Test
import kotlin.test.assertEquals

class SkillsRepositoryTest {
    
    private val localDataSource = mockk<SkillsLocalDataSource>()
    private val remoteDataSource = mockk<SkillsRemoteDataSource>()
    
    private val repository = SkillsRepository(
        localDataSource = localDataSource,
        remoteDataSource = remoteDataSource
    )
    
    @Test
    fun `getSkills returns cached data when available`() = runTest {
        // Arrange
        val expectedSkills = listOf(
            Skill(id = "1", name = "Kotlin"),
            Skill(id = "2", name = "Compose")
        )
        
        every { localDataSource.getAllSkills() } returns expectedSkills
        
        // Act
        val result = repository.getSkills()
        
        // Assert
        assertEquals(expectedSkills, result)
        verify(exactly = 0) { remoteDataSource.getSkills() } // Network не вызывается
    }
    
    @Test
    fun `getSkills fetches from network when cache is empty`() = runTest {
        // Arrange
        val expectedSkills = listOf(
            Skill(id = "1", name = "Kotlin")
        )
        
        every { localDataSource.getAllSkills() } returns emptyList()
        every { remoteDataSource.getSkills() } returns expectedSkills
        
        // Act
        val result = repository.getSkills()
        
        // Assert
        assertEquals(expectedSkills, result)
        verify(exactly = 1) { remoteDataSource.getSkills() }
        verify(exactly = 1) { localDataSource.saveSkills(expectedSkills) }
    }
    
    @Test
    fun `getSkillById returns null when skill not found`() = runTest {
        // Arrange
        every { localDataSource.getSkillById(any()) } returns null
        
        // Act
        val result = repository.getSkillById("nonexistent")
        
        // Assert
        assertEquals(null, result)
    }
    
    @Test
    fun `createSkill saves to both local and remote`() = runTest {
        // Arrange
        val newSkill = Skill(
            id = "3", 
            name = "KMP", 
            description = "Kotlin Multiplatform"
        )
        
        every { remoteDataSource.createSkill(any()) } returns newSkill
        
        // Act
        val result = repository.createSkill(
            name = "KMP",
            description = "Kotlin Multiplatform"
        )
        
        // Assert
        assertEquals(newSkill, result)
        verify(exactly = 1) { localDataSource.saveSkill(newSkill) }
    }
}

// Mock реализации для тестов
class SkillsLocalDataSource {
    fun getAllSkills(): List<Skill> = emptyList()
    suspend fun getSkillById(id: String): Skill? = null
    suspend fun saveSkill(skill: Skill) {}
    suspend fun saveSkills(skills: List<Skill>) {}
}

class SkillsRemoteDataSource {
    suspend fun getSkills(): List<Skill> = emptyList()
    suspend fun createSkill(skill: Skill): Skill = skill
}

data class Skill(
    val id: String,
    val name: String,
    val description: String? = null
)

class SkillsRepository(
    private val localDataSource: SkillsLocalDataSource,
    private val remoteDataSource: SkillsRemoteDataSource
) {
    suspend fun getSkills(): List<Skill> {
        val cached = localDataSource.getAllSkills()
        return if (cached.isNotEmpty()) {
            cached
        } else {
            val remote = remoteDataSource.getSkills()
            localDataSource.saveSkills(remote)
            remote
        }
    }
    
    suspend fun getSkillById(id: String): Skill? {
        return localDataSource.getSkillById(id)
    }
    
    suspend fun createSkill(name: String, description: String?): Skill {
        val skill = Skill(id = "new", name = name, description = description)
        val created = remoteDataSource.createSkill(skill)
        localDataSource.saveSkill(created)
        return created
    }
}
```

### Шаг 3: Тестирование корутин и StateFlow

Создайте `shared/src/commonTest/kotlin/ui/SkillsViewModelTest.kt`:

```kotlin
package ui

import app.cash.turbine.test
import io.mockk.*
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.test.runTest
import kotlin.test.Test
import kotlin.test.assertEquals

class SkillsViewModelTest {
    
    private val repository = mockk<SkillsRepository>()
    
    @Test
    fun `loadSkills updates UI state`() = runTest {
        // Arrange
        val expectedSkills = listOf(
            Skill(id = "1", name = "Kotlin"),
            Skill(id = "2", name = "Compose")
        )
        
        every { repository.getSkills() } returns expectedSkills
        
        val viewModel = SkillsViewModel(repository)
        
        // Act
        viewModel.loadSkills()
        
        // Assert
        assertEquals(expectedSkills, viewModel.skills.value)
        assertEquals(false, viewModel.isLoading.value)
        assertEquals(null, viewModel.error.value)
    }
    
    @Test
    fun `loadSkills sets error when repository fails`() = runTest {
        // Arrange
        val exception = Exception("Network error")
        every { repository.getSkills() } throws exception
        
        val viewModel = SkillsViewModel(repository)
        
        // Act
        viewModel.loadSkills()
        
        // Assert
        assertEquals(emptyList(), viewModel.skills.value)
        assertEquals(false, viewModel.isLoading.value)
        assertEquals(exception.message, viewModel.error.value?.message)
    }
    
    @Test
    fun `isLoading is true during loading`() = runTest {
        // Arrange
        every { repository.getSkills() } returns emptyList()
        
        val viewModel = SkillsViewModel(repository)
        
        // Act & Assert
        viewModel.isLoading.test {
            assertEquals(false, awaitItem()) // Initial state
            
            viewModel.loadSkills()
            
            assertEquals(true, awaitItem()) // Loading started
            assertEquals(false, awaitItem()) // Loading finished
            
            cancelAndIgnoreRemainingEvents()
        }
    }
}

class SkillsViewModel(
    private val repository: SkillsRepository
) {
    private val _skills = MutableStateFlow(emptyList<Skill>())
    val skills: StateFlow<List<Skill>> = _skills.asStateFlow()
    
    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()
    
    private val _error = MutableStateFlow<Exception?>(null)
    val error: StateFlow<Exception?> = _error.asStateFlow()
    
    suspend fun loadSkills() {
        _isLoading.value = true
        try {
            val skills = repository.getSkills()
            _skills.value = skills
            _error.value = null
        } catch (e: Exception) {
            _error.value = e
        } finally {
            _isLoading.value = false
        }
    }
}
```

---

## 3. Практика: Integration-тесты для платформенного кода

### Шаг 1: Integration-тесты для Android

Создайте `shared/src/androidUnitTest/kotlin/data/DatabaseIntegrationTest.kt`:

```kotlin
package data

import android.content.Context
import androidx.room.Room
import app.cash.sqldelight.db.SqlDriver
import com.skillsync.database.AppDatabase
import kotlinx.coroutines.test.runTest
import org.junit.After
import org.junit.Before
import org.junit.Test
import org.junit.Assert.*
import org.junit.Rule
import org.junit.rules.TemporaryFolder

class DatabaseIntegrationTest {
    
    @get:Rule
    val tempFolder = TemporaryFolder()
    
    private lateinit var database: AppDatabase
    private lateinit var driver: SqlDriver
    
    @Before
    fun setup() {
        // Создаем in-memory базу для тестов
        driver = TestSqlDriverFactory.createDriver()
        database = AppDatabase(driver)
    }
    
    @After
    fun teardown() {
        driver.close()
    }
    
    @Test
    fun `saveSkill and getAllSkills works correctly`() = runTest {
        // Arrange
        val skill = SkillEntity(
            id = "1",
            name = "Kotlin",
            description = "Multiplatform language"
        )
        
        // Act
        database.skillsQueries.insertSkill(
            id = skill.id,
            name = skill.name,
            description = skill.description
        )
        
        val result = database.skillsQueries.getAllSkills().executeAsList()
        
        // Assert
        assertEquals(1, result.size)
        assertEquals("Kotlin", result[0].name)
    }
    
    @Test
    fun `getSkillById returns correct skill`() = runTest {
        // Arrange
        database.skillsQueries.insertSkill(
            id = "1",
            name = "Kotlin",
            description = null
        )
        
        // Act
        val result = database.skillsQueries.getSkillById("1").executeAsOne()
        
        // Assert
        assertEquals("Kotlin", result.name)
    }
}

// Test driver для SQLDelight
class TestSqlDriverFactory {
    companion object {
        fun createDriver(): SqlDriver {
            return TestSqlDriver(
                schema = com.skillsync.database.Schema,
                name = "test.db"
            )
        }
    }
}
```

### Шаг 2: Integration-тесты для iOS (XCTest)

Создайте `shared/iosTest/SkillsRepositoryTests.swift`:

```swift
import XCTest
@testable import shared

class SkillsRepositoryTests: XCTestCase {
    
    var repository: SkillsRepository!
    var localDataSource: MockSkillsLocalDataSource!
    var remoteDataSource: MockSkillsRemoteDataSource!
    
    override func setUp() {
        super.setUp()
        
        localDataSource = MockSkillsLocalDataSource()
        remoteDataSource = MockSkillsRemoteDataSource()
        
        repository = SkillsRepository(
            localDataSource: localDataSource,
            remoteDataSource: remoteDataSource
        )
    }
    
    func testGetSkillsReturnsCachedData() async {
        // Arrange
        let expectedSkills = [
            Skill(id: "1", name: "Kotlin"),
            Skill(id: "2", name: "Compose")
        ]
        
        localDataSource.mockedGetAllSkills = { expectedSkills }
        
        // Act
        let result = await repository.getSkills()
        
        // Assert
        XCTAssertEqual(expectedSkills, result)
        XCTAssertTrue(remoteDataSource.getSkillsCalled == false)
    }
    
    func testGetSkillsFetchesFromNetworkWhenCacheIsEmpty() async {
        // Arrange
        let expectedSkills = [Skill(id: "1", name: "Kotlin")]
        
        localDataSource.mockedGetAllSkills = { [] }
        remoteDataSource.mockedGetSkills = { expectedSkills }
        
        // Act
        let result = await repository.getSkills()
        
        // Assert
        XCTAssertEqual(expectedSkills, result)
        XCTAssertTrue(remoteDataSource.getSkillsCalled == true)
    }
}

// Mock реализации для iOS тестов
class MockSkillsLocalDataSource: SkillsLocalDataSourceProtocol {
    var mockedGetAllSkills: () -> [Skill] = { [] }
    
    func getAllSkills() -> [Skill] {
        return mockedGetAllSkills()
    }
}

class MockSkillsRemoteDataSource: SkillsRemoteDataSourceProtocol {
    var getSkillsCalled = false
    var mockedGetSkills: () -> [Skill] = { [] }
    
    func getSkills() async -> [Skill] {
        getSkillsCalled = true
        return mockedGetSkills()
    }
}
```

---

## 4. Практика: UI-тесты (E2E)

### Шаг 1: UI-тесты для Android (Espresso)

Создайте `apps/skillsync-android/app/src/androidTest/kotlin/ui/SkillsScreenTest.kt`:

```kotlin
package ui

import androidx.test.core.app.ActivityScenario
import androidx.test.espresso.Espresso.onView
import androidx.test.espresso.assertion.ViewAssertions.matches
import androidx.test.espresso.matcher.ViewMatchers.*
import org.junit.Test

class SkillsScreenTest {
    
    @Test
    fun testSkillsListDisplaysItems() {
        // Запускаем приложение
        ActivityScenario.launch(MainActivity::class.java)
        
        // Ожидаем загрузки данных
        Thread.sleep(3000)
        
        // Проверяем что список не пустой
        onView(withText("Kotlin")).check(matches(isDisplayed()))
        onView(withText("Compose")).check(matches(isDisplayed()))
    }
    
    @Test
    fun testAddSkillButtonOpensDialog() {
        ActivityScenario.launch(MainActivity::class.java)
        
        // Нажимаем кнопку добавления навыка
        onView(withId(R.id.add_skill_button)).perform(click())
        
        // Проверяем что диалог открылся
        onView(withText("Add Skill")).check(matches(isDisplayed()))
    }
    
    @Test
    fun testSearchFilterSkills() {
        ActivityScenario.launch(MainActivity::class.java)
        
        // Вводим текст в поиск
        onView(withId(R.id.search_edit_text)).perform(typeText("Kotlin"))
        
        // Проверяем что отфильтровались только Kotlin-навыки
        onView(withText("Kotlin")).check(matches(isDisplayed()))
    }
}
```

### Шаг 2: UI-тесты для iOS (XCUITest)

Создайте `apps/skillsync-ios/SkillSyncTests/SkillsScreenTests.swift`:

```swift
import XCTest

class SkillsScreenTests: XCTestCase {
    
    var app: XCUIApplication!
    
    override func setUp() {
        super.setUp()
        
        continueAfterFailure = false
        
        app = XCUIApplication()
        app.launch()
    }
    
    func testSkillsListDisplaysItems() throws {
        // Ожидаем загрузки данных
        let skillsPredicate = NSPredicate(format: "label CONTAINS 'Kotlin'")
        let skillsQuery = app.staticTexts.matching(skillsPredicate)
        
        XCTAssertTrue(skillsQuery.waitForExistence(timeout: 5))
    }
    
    func testAddSkillButtonOpensDialog() throws {
        let addButton = app.buttons["Add Skill"]
        XCTAssertTrue(addButton.waitForExistence(timeout: 2))
        
        addButton.tap()
        
        let dialog = app.staticTexts["Add Skill"]
        XCTAssertTrue(dialog.waitForExistence(timeout: 2))
    }
    
    func testSearchFilterSkills() throws {
        let searchField = app.textFields["Search"]
        XCTAssertTrue(searchField.waitForExistence(timeout: 2))
        
        searchField.tap()
        searchField.typeText("Kotlin")
        
        let kotlinSkill = app.staticTexts["Kotlin"]
        XCTAssertTrue(kotlinSkill.waitForExistence(timeout: 3))
    }
}
```

---

## 5. Практика: Настройка CI/CD пайплайнов

### Шаг 1: GitHub Actions для KMP

Создайте `.github/workflows/ci.yml`:

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test-common:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Java
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'adopt'
      
      - name: Setup Kotlin
        uses: kotlin/kotlin-action@v3
        with:
          version: '1.9.0'
      
      - name: Cache Gradle
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*') }}
      
      - name: Run Common Tests
        run: ./gradlew :shared:test
        
      - name: Upload Test Results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results-common
          path: shared/build/reports/tests/

  test-android:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Java
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'adopt'
      
      - name: Run Android Tests
        run: ./gradlew :apps:skillsync-android:testDebugUnitTest
      
      - name: Run Android UI Tests
        run: ./gradlew :apps:skillsync-android:connectedCheck

  test-ios:
    runs-on: macos-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Run iOS Tests
        run: ./gradlew :shared:iosSimulatorArm64Test
      
      - name: Build iOS App
        run: ./gradlew :apps:skillsync-ios:assembleDebug

  code-quality:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Java
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'adopt'
      
      - name: Run Detekt (Kotlin linter)
        run: ./gradlew :shared:detekt
      
      - name: Run Ktlint
        run: ./gradlew :shared:ktlintCheck
      
      - name: Upload Quality Report
        uses: actions/upload-artifact@v3
        with:
          name: quality-report
          path: shared/build/reports/

  build-release:
    needs: [test-common, test-android, test-ios, code-quality]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Java
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'adopt'
      
      - name: Build Android Release
        run: ./gradlew :apps:skillsync-android:assembleRelease
      
      - name: Upload APK
        uses: actions/upload-artifact@v3
        with:
          name: android-release
          path: apps/skillsync-android/app/build/outputs/apk/release/

  deploy:
    needs: build-release
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Deploy to Production
        run: |
          echo "Deploying to production..."
          # Здесь будет ваша логика деплоя
```

### Шаг 2: Настройка Detekt для code quality

Создайте `shared/detekt-config.yml`:

```yaml
config:
  builds:
    excludedPaths:
      - '.*Test\.kt'

processors:
  active: true

console-reports:
  active: true

complexity:
  active: true
  LongMethod:
    active: true
    threshold: 60
  LongParameterList:
    active: true
    functionThreshold: 7
    constructorThreshold: 8

style:
  active: true
  MagicNumber:
    active: false # Отключаем для KMP (константы платформ)
  WildcardImport:
    active: true
    excludeImports:
      - 'kotlinx.*'

coroutines:
  active: true
  SuspendFunWithFlowReturnType:
    active: true
```

---

## 📝 Домашнее задание (Модуль 6)

### Задача: Построить полную пирамиду тестирования для SkillSync

**Требования:**

1. **Unit-тесты (commonTest):**
   - Напишите unit-тесты для Repository, ViewModel, Use Cases
   - Используйте MockK для моков зависимостей
   - Достижение: 70%+ coverage для commonMain

2. **Integration-тесты:**
   - Android: Напишите тесты для SQLDelight базы данных
   - iOS: Напишите XCTest тесты для платформенного кода
   - Протестируйте взаимодействие Repository + DataSource

3. **UI-тесты:**
   - Android: Напишите Espresso тесты для критичных сценариев
   - iOS: Напишите XCUITest тесты для тех же сценариев
   - Сценарии: Добавление навыка, поиск, фильтрация

4. **CI/CD:**
   - Настройте GitHub Actions пайплайн
   - Автоматический запуск тестов при push/PR
   - Сборка release APK/IPA на main ветке

5. **Code Quality:**
   - Настройте Detekt для Kotlin кода
   - Настройте SwiftLint для iOS кода (если есть)
   - Добавьте проверку в CI пайплайн

**Критерии сдачи:**
- ✅ 70%+ coverage для commonMain (отчет из IntelliJ/Xcode)
- ✅ Все unit-тесты проходят в CI
- ✅ Integration-тесты покрывают работу с базой данных
- ✅ UI-тесты проходят на эмуляторах/simulators
- ✅ CI пайплайн настроен и работает

**Бонусные задания:**
- Настройте mutation testing (Stryker/Mutant)
- Добавьте визуальные регрессионные тесты (Perceptual Diff)
- Настройте автоматический деплой на TestFlight/Play Store Internal Testing

---

## 💡 Советы по выполнению

1. **Начните с unit-тестов:** Они самые быстрые и дешевые.
2. **Тестируйте поведение, не реализацию:** Тесты должны проверять что делает код, а не как.
3. **Используйте Arrange-Act-Assert:** Структурируйте тесты для читаемости.
4. **Не тестируйте тривиальное:** Геттеры/сеттеры не требуют тестов.
5. **Автоматизируйте всё:** CI/CD должен запускать все тесты автоматически.

---

**Следующий модуль:** В Module_07_Architecture_Patterns мы изучим Clean Architecture, MVI, Modularization и создание масштабируемой архитектуры.

Удачи! 🚀

# 📘 Модуль 8: Advanced Testing Strategies

**В этом модуле вы освоите comprehensive testing для KMP: unit tests, integration tests, UI tests с Compose Testings и тестирование expect/actual.**

**Цели модуля:**
1. Настроить multiplatform test infrastructure
2. Написать comprehensive unit tests для common code
3. Реализовать integration tests с тестовыми дублями
4. Настроить UI testing для Compose Multiplatform

**Время выполнения:** ~35 часов.

---

## 1. Test Infrastructure Setup

### Multiplatform Test Configuration:

```kotlin
// shared/build.gradle.kts - Test configuration
kotlin {
    sourceSets {
        val commonTest by getting {
            dependencies {
                // Kotlin Test framework
                implementation(kotlin("test"))
                
                // Mocking library (MockK)
                implementation("io.mockk:mockk:1.13.4")
                
                // Turbine for Flow testing
                implementation("app.cash.turbine:turbine:1.0.0")
                
                // Coroutines test utilities
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
            }
        }
        
        val androidUnitTest by getting {
            dependencies {
                implementation(kotlin("test-junit"))
                implementation("junit:junit:4.13.2")
            }
        }
        
        val iosTest by getting {
            dependencies {
                implementation(kotlin("test"))
            }
        }
    }
}

// Test tasks configuration
tasks.withType<Test> {
    useJUnit()
    
    testLogging {
        events("passed", "skipped", "failed")
        showStandardStreams = true
    }
}

// Compose testing for Android
tasks.named("androidUnitTest") {
    jvmArgs("--add-opens", "java.base/java.lang=ALL-UNNAMED")
}
```

---

## 2. Unit Tests для Common Code

### Testing Use Cases:

```kotlin
// commonMain - Use case to test
class GetSkillsUseCase(
    private val skillsRepository: SkillsRepository
) {
    suspend operator fun invoke(categoryId: String? = null): Result<List<Skill>> {
        return try {
            val skills = if (categoryId != null) {
                skillsRepository.getSkillsByCategory(categoryId)
            } else {
                skillsRepository.getAllSkills()
            }
            
            Result.success(skills)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// commonTest - Unit tests
class GetSkillsUseCaseTest {
    
    private val mockRepository = mock<SkillsRepository>()
    private lateinit var useCase: GetSkillsUseCase
    
    @BeforeTest
    fun setup() {
        useCase = GetSkillsUseCase(mockRepository)
    }
    
    @Test
    fun `returns all skills when no category specified`() = runTest {
        // Arrange
        val expectedSkills = listOf(
            Skill("1", "Kotlin", "programming", SkillLevel.BEGINNER),
            Skill("2", "Swift", "programming", SkillLevel.INTERMEDIATE)
        )
        
        every { mockRepository.getAllSkills() } returns expectedSkills
        
        // Act
        val result = useCase()
        
        // Assert
        assertEquals(Result.success(expectedSkills), result)
        verify { mockRepository.getAllSkills() }
    }
    
    @Test
    fun `returns filtered skills when category specified`() = runTest {
        // Arrange
        val expectedSkills = listOf(
            Skill("1", "Kotlin", "programming", SkillLevel.BEGINNER)
        )
        
        every { mockRepository.getSkillsByCategory("programming") } returns expectedSkills
        
        // Act
        val result = useCase(categoryId = "programming")
        
        // Assert
        assertEquals(Result.success(expectedSkills), result)
        verify { mockRepository.getSkillsByCategory("programming") }
    }
    
    @Test
    fun `returns error when repository throws exception`() = runTest {
        // Arrange
        val expectedError = Exception("Database error")
        every { mockRepository.getAllSkills() } throws expectedError
        
        // Act
        val result = useCase()
        
        // Assert
        assertTrue(result.isFailure)
        assertEquals(expectedError, result.exceptionOrNull())
    }
}
```

### Testing StateFlow и SharedFlow:

```kotlin
// commonMain - ViewModel to test
class SkillsViewModel(
    private val getSkillsUseCase: GetSkillsUseCase
) : ViewModel() {
    
    private val _state = MutableStateFlow(SkillsUiState())
    val state: StateFlow<SkillsUiState> = _state.asStateFlow()
    
    private val _events = MutableSharedFlow<UiEvent>()
    val events: SharedFlow<UiEvent> = _events
    
    init {
        viewModelScope.launch {
            loadSkills()
        }
    }
    
    fun loadSkills() = viewModelScope.launch {
        _state.value = _state.value.copy(isLoading = true)
        
        when (val result = getSkillsUseCase()) {
            is Result.Success -> {
                _state.value = _state.value.copy(
                    skills = result.data,
                    isLoading = false
                )
            }
            is Result.Failure -> {
                _state.value = _state.value.copy(
                    error = result.exception.message,
                    isLoading = false
                )
            }
        }
    }
}

// commonTest - ViewModel tests
class SkillsViewModelTest {
    
    private val mockUseCase = mock<GetSkillsUseCase>()
    private lateinit var viewModel: SkillsViewModel
    
    @BeforeTest
    fun setup() {
        viewModel = SkillsViewModel(mockUseCase)
    }
    
    @Test
    fun `state shows loading initially`() = runTest {
        // Assert initial state
        assertEquals(true, viewModel.state.value.isLoading)
    }
    
    @Test
    fun `state shows skills after successful load`() = runTest {
        // Arrange
        val expectedSkills = listOf(
            Skill("1", "Kotlin", "programming", SkillLevel.BEGINNER)
        )
        
        every { mockUseCase() } returns Result.success(expectedSkills)
        
        // Wait for state to update
        advanceUntilIdle()
        
        // Assert
        val state = viewModel.state.value
        assertEquals(false, state.isLoading)
        assertEquals(expectedSkills, state.skills)
        assertNull(state.error)
    }
    
    @Test
    fun `state shows error after failed load`() = runTest {
        // Arrange
        val expectedError = Exception("Network error")
        every { mockUseCase() } returns Result.failure(expectedError)
        
        // Wait for state to update
        advanceUntilIdle()
        
        // Assert
        val state = viewModel.state.value
        assertEquals(false, state.isLoading)
        assertEquals(expectedError.message, state.error)
    }
}

// Test extension for StateFlow
fun <T> StateFlow<T>.test(): Turbine<T> = this.test()
```

---

## 3. Integration Tests с Test Doubles

### Fake Implementations:

```kotlin
// commonTest - Fake repository for integration tests
class FakeSkillsRepository(
    private val skills: MutableList<Skill> = mutableListOf()
) : SkillsRepository {
    
    override suspend fun getAllSkills(): List<Skill> = skills.toList()
    
    override suspend fun getSkillById(id: String): Skill? = 
        skills.find { it.id == id }
    
    override suspend fun getSkillsByCategory(categoryId: String): List<Skill> = 
        skills.filter { it.categoryId == categoryId }
    
    override suspend fun addSkill(skill: Skill): Result<Unit> {
        skills.add(skill)
        return Result.success(Unit)
    }
    
    override suspend fun updateSkill(skill: Skill): Result<Unit> {
        val index = skills.indexOfFirst { it.id == skill.id }
        if (index != -1) {
            skills[index] = skill
            return Result.success(Unit)
        }
        return Result.failure(Exception("Skill not found"))
    }
    
    override suspend fun deleteSkill(id: String): Result<Unit> {
        val removed = skills.removeAll { it.id == id }
        return if (removed) Result.success(Unit) 
        else Result.failure(Exception("Skill not found"))
    }
}

// Integration test
class SkillsRepositoryIntegrationTest {
    
    private lateinit var repository: FakeSkillsRepository
    
    @BeforeTest
    fun setup() {
        repository = FakeSkillsRepository(
            mutableListOf(
                Skill("1", "Kotlin", "programming", SkillLevel.BEGINNER),
                Skill("2", "Swift", "programming", SkillLevel.INTERMEDIATE),
                Skill("3", "Figma", "design", SkillLevel.ADVANCED)
            )
        )
    }
    
    @Test
    fun `getAllSkills returns all skills`() = runTest {
        val result = repository.getAllSkills()
        
        assertEquals(3, result.size)
    }
    
    @Test
    fun `getSkillsByCategory filters correctly`() = runTest {
        val result = repository.getSkillsByCategory("programming")
        
        assertEquals(2, result.size)
        assertTrue(result.all { it.categoryId == "programming" })
    }
    
    @Test
    fun `addSkill adds new skill`() = runTest {
        val newSkill = Skill("4", "React", "frontend", SkillLevel.BEGINNER)
        
        val result = repository.addSkill(newSkill)
        
        assertTrue(result.isSuccess)
        assertEquals(4, repository.getAllSkills().size)
    }
    
    @Test
    fun `updateSkill updates existing skill`() = runTest {
        val updatedSkill = Skill(
            id = "1", 
            name = "Kotlin Advanced", 
            categoryId = "programming", 
            level = SkillLevel.ADVANCED
        )
        
        val result = repository.updateSkill(updatedSkill)
        
        assertTrue(result.isSuccess)
        assertEquals("Kotlin Advanced", repository.getSkillById("1")?.name)
    }
    
    @Test
    fun `deleteSkill removes skill`() = runTest {
        val result = repository.deleteSkill("1")
        
        assertTrue(result.isSuccess)
        assertEquals(2, repository.getAllSkills().size)
    }
}
```

---

## 4. UI Testing с Compose Test

### Android UI Tests:

```kotlin
// androidTest - Compose UI tests
class SkillsScreenTest {
    
    @get:Rule
    val composeTestRule = createComposeRule()
    
    @Test
    fun `displays skills list`() {
        // Setup test data and navigation
        val viewModel = SkillsViewModel(FakeSkillsRepository())
        
        composeTestRule.setContent {
            SkillsScreen(viewModel = viewModel)
        }
        
        // Wait for content to load
        composeTestRule.waitForIdle()
        
        // Assert skills are displayed
        composeTestRule.onNodeWithText("Kotlin").assertIsDisplayed()
        composeTestRule.onNodeWithText("Swift").assertIsDisplayed()
    }
    
    @Test
    fun `shows loading indicator initially`() {
        composeTestRule.setContent {
            SkillsScreen(viewModel = SkillsViewModel(FakeSkillsRepository()))
        }
        
        // Assert loading indicator is shown
        composeTestRule.onNodeWithContentDescription("Loading")
            .assertIsDisplayed()
    }
    
    @Test
    fun `clicking refresh button triggers reload`() {
        val viewModel = SkillsViewModel(FakeSkillsRepository())
        
        composeTestRule.setContent {
            SkillsScreen(viewModel = viewModel)
        }
        
        // Click refresh button
        composeTestRule.onNodeWithContentDescription("Refresh")
            .performClick()
        
        // Verify refresh was triggered (check state changes)
        composeTestRule.waitForIdle()
    }
}

// iOS UI Tests (XCUITest) - expect/actual pattern
// iosTest
expect fun createUiTestRunner(): UiTestRunner

// androidTest
actual fun createUiTestRunner(): UiTestRunner {
    return AndroidUiTestRunner()
}

// iosTest  
actual fun createUiTestRunner(): UiTestRunner {
    return IosUiTestRunner()
}

// Common UI test logic
interface UiTestRunner {
    fun launchApp()
    fun tapOn(element: String)
    fun assertElementVisible(element: String)
}

class SkillsUiTest {
    
    private val testRunner = createUiTestRunner()
    
    @Test
    fun `user can view skills list`() {
        testRunner.launchApp()
        
        // Navigate to skills screen
        testRunner.tapOn("Skills")
        
        // Assert skills are visible
        testRunner.assertElementVisible("Kotlin")
        testRunner.assertElementVisible("Swift")
    }
}
```

---

## 5. Test Coverage & Quality Metrics

### Jacoco Configuration для Android:

```kotlin
// android/build.gradle.kts
plugins {
    id("jacoco")
}

tasks.register<JacocoReport>("createJacocoReport") {
    group = "verification"
    description = "Creates Jacoco coverage report for Android unit tests"
    
    // Configure input files
    classDirectories.setFrom(
        fileTree("build/intermediates/javac/debug/classes"),
        fileTree("build/tmp/kotlin-classes/debug")
    )
    
    sourceDirectories.setFrom(
        fileTree("src/main/java"),
        fileTree("src/main/kotlin")
    )
    
    executionData.setFrom(
        fileTree("build/jacoco").files.filter { it.exists() }
    )
    
    reports {
        html.required.set(true)
        xml.required.set(true)
    }
}

// Enable Jacoco for test tasks
tasks.withType<Test> {
    isIgnoreFailures = true
    
    finalizedBy("createJacocoReport")
}
```

### Test Coverage Report Task:

Создайте `scripts/generate-coverage-report.sh`:

```bash
#!/bin/bash

echo "📊 Generating test coverage report..."

# Clean previous reports
rm -rf build/reports/jacoco

# Run tests with coverage
./gradlew testDebugUnitTest createJacocoReport --continue

# Open HTML report
if [ "$(uname)" == "Darwin" ]; then
    open build/reports/jacoco/html/index.html
elif command -v xdg-open &> /dev/null; then
    xdg-open build/reports/jacoco/html/index.html
fi

echo "✅ Coverage report generated!"
```

---

## 📝 Домашнее задание (Модуль 8)

### Задача: Comprehensive test suite для SkillSync

**Требования:**
1. Напишите unit tests для всех use cases (минимум 80% coverage)
2. Создайте fake implementations для integration tests
3. Напишите UI tests для основных экранов (Compose Test)
4. Настройте Jacoco coverage reporting

**Критерии сдачи:**
- ✅ Unit tests для 100% use cases
- ✅ Integration tests с fake implementations
- ✅ UI tests для 3+ основных экранов
- ✅ Coverage report показывает >70% coverage

---

**Следующий модуль:** В Module_09 мы изучим CI/CD pipeline для KMP приложений.

Удачи! 🚀

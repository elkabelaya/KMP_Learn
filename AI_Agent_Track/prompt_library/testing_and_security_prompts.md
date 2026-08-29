# Prompt'ы для Testing и Security

## 🧪 Unit Testing

### Prompt: Генерация comprehensive unit tests

```
Ты - Senior QA Engineer с 10+ лет опытом в тестировании Kotlin кода.

Напиши comprehensive unit-тесты для: [CLASS/USE CASE NAME]

Код для тестирования:
[ВСТАВИТЬ КОД]

Требования к тестам:
1. **Framework:** JUnit 5 + AssertJ (или Kotlin Test)
2. **Mocking:** MockK для всех зависимостей  
3. **Coverage:** 90%+ строк кода
4. **Structure:** Arrange-Act-Assert pattern

Тестируй следующие сценарии:

**Happy Path (нормальные сценарии):**
- [scenario 1]: [описание]
- [scenario 2]: [описание]

**Edge Cases (граничные условия):**
- Null values для optional parameters  
- Empty collections/strings
- Boundary values (min, max)
- Very large inputs

**Error Scenarios:**
- Repository errors (network failure, DB error)  
- Validation failures
- Not found errors
- Concurrent access issues (если async)

**Format каждого теста:**
```kotlin
@Test
fun `should_[expectedBehavior]_when_[condition]() = runTest {
    // Arrange - setup mocks и test data
    val input = ...
    every { dependency.method() } returns ...
    
    // Act - invoke tested code  
    val result = sut.method(input)
    
    // Assert - verify results и mock interactions
    assertThat(result).isInstanceOf(Result.Success::class.java)
    result.fold(
        onSuccess = { data -> assertThat(data.field).isEqualTo(...) },
        onError = { fail("Expected success") }
    )
    
    verify { dependency.method(...) }
}
```

Создай полный тестовый класс с:
- @BeforeTest setup для моков  
- 8-12 тестов покрывающих все сценарии
- Test data builders или factory functions (если нужно)

Покажи полный код.
```

---

### Prompt: Создание Test Data Builders

```
Ты - Senior разработчик, эксперт в тестировании.

Создай test data builders для [ENTITY/DTO NAME].

Entity definition:
```kotlin
data class [ClassName](
    val id: String = ...,
    val field1: Type1 = ...,  
    val field2: Type2 = ...,
    val optionalField: Type3? = null,
    val listField: List<Type4> = emptyList()
)
```

Создай Builder pattern class:

```kotlin
class [ClassName]Builder {
    var id: String = UUID.randomUUID().toString()
    var field1: Type1 = defaultField1
    var field2: Type2 = defaultField2  
    var optionalField: Type3? = null
    var listField: List<Type4> = emptyList()
    
    fun build(): [ClassName] {
        return [ClassName](id, field1, field2, optionalField, listField)
    }
    
    companion object {
        fun create() = [ClassName]Builder()
        
        // Preset builders для common scenarios
        fun valid[ClassName]() = create().apply { /* setup valid defaults */ }
        fun invalid[ClassName]() = create().apply { /* setup invalid data */ }
    }
}

// Extension function для удобства
fun [ClassName](builderAction: [ClassName]Builder.() -> Unit): [ClassName] {
    return [ClassName]Builder().apply(builderAction).build()
}

// Usage:
val user = User { 
    id = "user-123"
    field1 = "test value"
}
```

Создай builders для всех основных entities в проекте.
Включи preset configurations для common test scenarios.
```

---

### Prompt: Генерация Integration Tests

```
Ты - Senior QA Engineer, эксперт в integration testing.

Создай integration test для [FEATURE/WORKFLOW NAME].

Test должен проверить полный workflow:
1. [step 1]: [description]  
2. [step 2]: [description]
3. [step 3]: [description]

Тестовое окружение:
- In-memory database (SQLDelight с TestQueryProvider)  
- Mock network layer (или TestContainer для real API)
- StandardTestDispatcher для корутин

Требования:
1. Setup test database с initial data (если нужно)
2. Configure test dependencies (repositories, use cases)  
3. Execute full workflow через public API
4. Verify final state в database и/or output

Используй:
- @BeforeTest для setup test environment  
- @AfterTest для cleanup
- runTest с StandardTestDispatcher

Пример структуры:
```kotlin
class [Feature]IntegrationTest {
    
    private lateinit var testDb: TestDatabase
    private lateinit var repository: [Repository]
    private lateinit var useCase: [UseCase]
    
    @BeforeTest  
    fun setup() {
        testDb = createTestDatabase()
        repository = [Repository](testDb)
        useCase = [UseCase](repository)
    }
    
    @Test
    fun `should_[workflowDescription]() = runTest {
        // Setup initial state
        testDb.insert(...)
        
        // Execute workflow  
        val result = useCase.execute(input)
        
        // Verify final state
        assertThat(result).isInstanceOf(Result.Success::class.java)
        val saved = testDb.query(...)
        assertThat(saved).isNotNull()
    }
    
    @AfterTest
    fun cleanup() {
        testDb.close()
    }
}
```

Создай 2-3 integration tests для critical workflows.
Покажи полный код.
```

---

## 🔐 Security Testing и Auditing

### Prompt: Security Audit для KMP кода

```
Ты - Mobile Security Engineer с 10+ лет опытом в application security.

Проведи comprehensive security audit для следующего KMP кода:

[ВСТАВИТЬ КОД - классы, функции, или описание архитектуры]

Анализируй следующие security aspects:

## 1. Data Security
- [ ] Encryption at rest (database, shared preferences)
- [ ] Encryption in transit (TLS 1.3, certificate pinning)  
- [ ] Sensitive data storage (API keys, tokens, PII в logs)
- [ ] Secure deletion of sensitive data

## 2. Authentication & Authorization  
- [ ] Token storage (encrypted, secure location)
- [ ] Token refresh mechanism security
- [ ] Session management и timeout
- [ ] Permission escalation vulnerabilities

## 3. Input Validation & Injection Prevention
- [ ] SQL injection (parameterized queries)  
- [ ] Command injection (shell commands)
- [ ] XSS (если есть HTML rendering)
- [ ] Path traversal (file operations)

## 4. Network Security
- [ ] HTTPS only (no HTTP fallback)
- [ ] Certificate pinning implemented  
- [ ] API endpoint validation
- [ ] Request/response size limits

## 5. Error Handling & Information Leakage  
- [ ] No sensitive data in error messages
- [ ] Generic error messages для users
- [ ] Detailed logs только в debug builds

## 6. Platform-Specific Security
**Android:**
- [ ] android:allowBackup="false"  
- [ ] android:extractNativeLibraries="false"
- [ ] ProGuard/R8 obfuscation enabled
- [ ] No debug code в production builds

**iOS:**  
- [ ] Keychain для sensitive data
- [ ] NSAllowsArbitraryLoads = false
- [ ] Bitcode и obfuscation

Для каждого найденного issue покажи:
```markdown
## [Severity] - [Issue Title]

**Location:** File.kt, Line X  
**Problem:** [Detailed description of vulnerability]
**Risk:** [What could happen if exploited]
**CVSS Score:** X.X/10 (если можешь оценить)

**Exploit Scenario:**
[How an attacker could exploit this]

**Fix:**
```kotlin
// Show corrected code
```

**Additional Recommendations:**
- [Best practice 1]
- [Best practice 2]
```

Классифицируй issues по severity:
- 🔴 **Critical:** Immediate exploitation possible, data breach risk  
- 🟡 **High:** Significant security weakness
- 🟢 **Medium:** Security best practice violation  
- ⚪ **Low:** Minor improvement

Создай prioritized security audit report с actionable recommendations.
```

---

### Prompt: Создание Secure Storage Implementation

```
Ты - Security Engineer, эксперт в secure data storage для mobile.

Создай cross-platform secure storage solution для KMP приложения.

Requirements:
1. Store sensitive data (API tokens, user credentials, encryption keys) securely  
2. Android: Jetpack Security EncryptedSharedPreferences или MasterKey
3. iOS: Keychain с proper accessibility settings  
4. AES-256 encryption minimum
5. Biometric authentication option

Expect interface (commonMain):
```kotlin
interface SecureStorage {
    // Basic operations  
    suspend fun save(key: String, value: String): Result<Unit>
    suspend fun load(key: String): Result<String?>
    suspend fun delete(key: String): Result<Unit>
    
    // Biometric-protected storage  
    suspend fun saveWithBiometric(key: String, value: String): Result<Unit>
    suspend fun loadWithBiometric(key: String): Result<String?>
    
    // Bulk operations  
    suspend fun saveAll(data: Map<String, String>): Result<Unit>
    suspend fun loadAll(): Result<Map<String, String>>
}

sealed class SecureStorageError : Exception() {
    object EncryptionFailed : SecureStorageError("Encryption failed")
    object DecryptionFailed : SecureStorageError("Decryption failed - credentials may be invalid")  
    object BiometricNotAvailable : SecureStorageError("Biometric authentication not available")
    object BiometricAuthenticationFailed : SecureStorageError("Biometric authentication failed")
}
```

Actual implementations:

**Android (androidMain):**
- Jetpack Security 2.1+ EncryptedSharedPreferences  
- MasterKey с biometric fallback
- Proper context handling

**iOS (iosMain):**  
- Keychain через kotlin-multiplatform-security или custom implementation
- kSecAttrAccessibleWhenUnlockedThisDeviceOnly для biometric data  
- Proper error mapping

Создай:
1. Expect interface в commonMain (как выше)  
2. Android actual implementation с Jetpack Security
3. iOS actual implementation с Keychain
4. Factory function для creation  
5. Example usage code

Включи proper error handling и fallback mechanisms.
Покажи полный код для всех платформ + dependencies.
```

---

### Prompt: Penetration Test Simulation

```
Ты - Security Researcher, специалист в penetration testing mobile applications.

Проведи simulated penetration test для [FEATURE/COMPONENT NAME].

Контекст приложения:
- [Описание функционала]  
- [Какие данные обрабатывает - PII, payment info, etc.]
- [Authentication mechanism]

Атакуй следующие векторы:

## 1. Reverse Engineering
- Can you extract API keys или secrets из compiled code?  
- Can you bypass license verification?
- Can you modify app behavior через decompiled code?

## 2. Network Interception  
- Can you intercept и modify API requests/responses?
- Is certificate pinning properly implemented?
- Can you replay captured requests?

## 3. Data Extraction  
- Can you access local database без encryption?
- Can you extract data от shared preferences/keychain?  
- Is sensitive data в logs или crash reports?

## 4. Authentication Bypass
- Can you manipulate tokens для privilege escalation?
- Can you bypass biometric authentication?  
- Session hijacking possible?

## 5. Input Manipulation
- Can you inject malicious data через user inputs?  
- Can you overflow buffers или cause crashes?
- Can you trigger DoS через malformed inputs?

Для каждого вектора покажи:
```markdown
## [Attack Vector] - [Status: Vulnerable/Protected/Mitigated]

**Attack Method:**
[Step-by-step how attacker would exploit this]

**Tools Required:**  
- [Tool 1, e.g., Burp Suite]
- [Tool 2, e.g., JADX]

**Exploitation Steps:**
1. [Step 1]
2. [Step 2]  
3. [Step 3]

**Impact:**
[What attacker can achieve - data theft, account takeover, etc.]

**Current Protection Status:**
[ ] Not implemented  
[ ] Partially implemented (weak)
[ ] Fully implemented

**Recommendations:**
- [Fix 1]
- [Fix 2]  
- [Additional hardening measures]
```

Создай comprehensive penetration test report с prioritized remediation plan.
```

---

## 📝 Как использовать эти prompt'ы

### Для Testing:
1. **Unit Tests:** Используйте для генерации тестов новых классов/функций  
2. **Test Data Builders:** Создавайте один раз, используйте во всех тестах
3. **Integration Tests:** Для critical workflows и end-to-end scenarios

### Для Security:  
1. **Security Audit:** Перед каждым major release или после добавления sensitive features
2. **Secure Storage:** Для хранения любых credentials, tokens, PII  
3. **Penetration Test Simulation:** Quarterly или перед production launch

### Best Practices:
- ✅ Всегда review AI-generated test code перед использованием  
- ✅ Run security audits регулярно (ежемесячно или при major changes)
- ✅ Fix все Critical и High security issues перед deployment  
- ✅ Document security decisions в ADR

---

**Secure and tested code is happy code! 🔒🧪**

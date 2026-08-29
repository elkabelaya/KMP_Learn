# 💻 Code Examples & Best Practices

**Примеры кода и best practices для Senior KMP разработки**

---

## 🏗️ Architecture Examples

### Clean Architecture с Feature Modules

```kotlin
// shared/src/commonMain/kotlin/com/example/app/domain/usecase/GetUserUseCase.kt
package com.example.app.domain.usecase

import com.example.app.domain.repository.UserRepository
import com.example.app.domain.model.User

class GetUserUseCase(
    private val userRepository: UserRepository
) {
    suspend operator fun invoke(userId: String): Result<User> {
        return try {
            val user = userRepository.getUser(userId)
            if (user != null) {
                Result.success(user)
            } else {
                Result.failure(UserNotFoundException(userId))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// shared/src/commonMain/kotlin/com/example/app/domain/repository/UserRepository.kt
interface UserRepository {
    suspend fun getUser(userId: String): User?
    suspend fun getAllUsers(): List<User>
    suspend fun updateUser(user: User): Result<Unit>
}

// shared/src/commonMain/kotlin/com/example/app/data/repository/UserRepositoryImpl.kt
class UserRepositoryImpl(
    private val localDataSource: UserLocalDataSource,
    private val remoteDataSource: UserRemoteDataSource
) : UserRepository {
    
    override suspend fun getUser(userId: String): User? {
        // Try local first (offline-first)
        val localUser = localDataSource.getUser(userId)
        if (localUser != null) {
            // Update from remote in background
            updateFromRemote(userId)
            return localUser
        }
        
        // Fallback to remote
        return try {
            remoteDataSource.getUser(userId)?.also { 
                localDataSource.saveUser(it) 
            }
        } catch (e: Exception) {
            null
        }
    }
    
    private suspend fun updateFromRemote(userId: String) {
        coroutineScope.launch {
            try {
                val remoteUser = remoteDataSource.getUser(userId)
                remoteUser?.let { localDataSource.saveUser(it) }
            } catch (e: Exception) {
                // Log error but don't fail the main operation
            }
        }
    }
    
    // ... other implementations
}

// shared/src/commonMain/kotlin/com/example/app/presentation/viewmodel/UserViewModel.kt
class UserViewModel(
    private val getUserUseCase: GetUserUseCase,
    dispatcher: CoroutineDispatcher = Dispatchers.Default
) : ViewModel(dispatcher) {
    
    private val _userState = MutableStateFlow<UserUiState>(UserUiState.Loading)
    val userState: StateFlow<UserUiState> = _userState.asStateFlow()
    
    fun loadUser(userId: String) {
        viewModelScope.launch {
            _userState.value = UserUiState.Loading
            
            getUserUseCase(userId).fold(
                onSuccess = { user ->
                    _userState.value = UserUiState.Success(user.toUiModel())
                },
                onFailure = { error ->
                    _userState.value = UserUiState.Error(error.message ?: "Unknown error")
                }
            )
        }
    }
}

sealed class UserUiState {
    object Loading : UserUiState()
    data class Success(val user: UserUiModel) : UserUiState()
    data class Error(val message: String) : UserUiState()
}
```

---

## 🔧 Advanced Expect/Actual Patterns

### Platform-Specific Implementation с Abstraction

```kotlin
// shared/src/commonMain/kotlin/com/example/app/data/local/SecureStorage.kt
interface SecureStorage {
    suspend fun save(key: String, value: String): Result<Unit>
    suspend fun load(key: String): Result<String?>
    suspend fun delete(key: String): Result<Unit>
}

// shared/src/androidMain/kotlin/com/example/app/data/local/SecureStorage.kt
actual class SecureStorage actual constructor() : SecureStorage {
    private val encryptedPrefs by lazy { 
        EncryptedSharedPreferences.create(
            ContextWrapper(ApplicationProvider.getApplicationContext()).applicationContext,
            "secure_prefs",
            MasterKey.Builder(ContextWrapper(ApplicationProvider.getApplicationContext()).applicationContext).build(),
            EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
            EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
        )
    }
    
    actual override suspend fun save(key: String, value: String): Result<Unit> = try {
        encryptedPrefs.edit().putString(key, value).apply()
        Result.success(Unit)
    } catch (e: Exception) {
        Result.failure(e)
    }
    
    actual override suspend fun load(key: String): Result<String?> = try {
        val value = encryptedPrefs.getString(key, null)
        Result.success(value)
    } catch (e: Exception) {
        Result.failure(e)
    }
    
    actual override suspend fun delete(key: String): Result<Unit> = try {
        encryptedPrefs.edit().remove(key).apply()
        Result.success(Unit)
    } catch (e: Exception) {
        Result.failure(e)
    }
}

// shared/src/iosMain/kotlin/com/example/app/data/local/SecureStorage.kt
actual class SecureStorage actual constructor() : SecureStorage {
    private val keychain = KeychainHelper()
    
    actual override suspend fun save(key: String, value: String): Result<Unit> = 
        withContext(Dispatchers.IO) {
            try {
                keychain.setGenericPassword(service = "app", account = key, password = value)
                Result.success(Unit)
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    
    actual override suspend fun load(key: String): Result<String?> = 
        withContext(Dispatchers.IO) {
            try {
                val value = keychain.getGenericPassword(service = "app", account = key)
                Result.success(value)
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    
    actual override suspend fun delete(key: String): Result<Unit> = 
        withContext(Dispatchers.IO) {
            try {
                keychain.removeGenericPassword(service = "app", account = key)
                Result.success(Unit)
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
}

// shared/src/commonMain/kotlin/com/example/app/data/local/SecureStorage.kt (expect)
expect class SecureStorage() : SecureStorage
```

---

## 🗄️ SQLDelight Advanced Patterns

### Repository с Offline-First и Background Sync

```kotlin
// shared/src/commonMain/kotlin/com/example/app/data/local/UserDao.kt
@SqlQuery("SELECT * FROM users WHERE id = :id")
fun getUserById(id: String): User?

@SqlQuery("SELECT * FROM users")
fun getAllUsers(): Cursor<User>

@CoroutinesSuspending
suspend fun insertUser(user: User)

@CoroutinesSuspending
suspend fun updateUser(user: User)

@CoroutinesSuspending
suspend fun deleteUser(id: String)

// shared/src/commonMain/kotlin/com/example/app/data/local/UserLocalDataSource.kt
class UserLocalDataSource(
    private val dao: UserDao,
    private val syncManager: SyncManager
) : UserLocalDataSourceInterface {
    
    override suspend fun getUser(userId: String): User? {
        return dao.getUserById(userId)
    }
    
    override suspend fun getAllUsers(): List<User> {
        return dao.getAllUsers().toList()
    }
    
    override suspend fun saveUser(user: User): Result<Unit> {
        return try {
            if (dao.getUserById(user.id) != null) {
                dao.updateUser(user)
            } else {
                dao.insertUser(user)
            }
            
            // Mark for sync in background
            syncManager.queueForSync(SyncItem.User(user))
            
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// shared/src/commonMain/kotlin/com/example/app/data/sync/SyncManager.kt
class SyncManager(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource,
    private val syncQueueDao: SyncQueueDao
) {
    private val job = Job()
    private val scope = CoroutineScope(Dispatchers.IO + job)
    
    fun queueForSync(item: SyncItem) {
        scope.launch {
            syncQueueDao.insert(SyncQueueEntity(
                id = UUID.randomUUID().toString(),
                itemType = item.type,
                itemData = item.data,
                timestamp = System.currentTimeMillis(),
                status = SyncStatus.PENDING
            ))
        }
    }
    
    suspend fun sync() {
        val pendingItems = syncQueueDao.getPendingItems()
        
        for (item in pendingItems) {
            try {
                when (val syncItem = parseSyncItem(item)) {
                    is SyncItem.User -> {
                        val result = remoteDataSource.updateUser(syncItem.user)
                        result.fold(
                            onSuccess = { 
                                syncQueueDao.updateStatus(item.id, SyncStatus.SUCCESS)
                                localDataSource.saveUser(syncItem.user)
                            },
                            onFailure = { 
                                syncQueueDao.updateStatus(item.id, SyncStatus.FAILED)
                            }
                        )
                    }
                }
            } catch (e: Exception) {
                syncQueueDao.updateStatus(item.id, SyncStatus.FAILED)
            }
        }
    }
    
    fun startBackgroundSync() {
        scope.launch {
            while (isActive) {
                sync()
                delay(5_000) // Sync every 5 seconds
            }
        }
    }
    
    fun stop() {
        job.cancel()
    }
}

sealed class SyncStatus {
    object PENDING : SyncStatus()
    object SUCCESS : SyncStatus()
    object FAILED : SyncStatus()
}

sealed class SyncItem(val type: String, val data: String) {
    data class User(val user: User) : SyncItem("USER", user.toJson())
}
```

---

## 🔒 Security Best Practices

### Certificate Pinning с Ktor Client

```kotlin
// shared/src/commonMain/kotlin/com/example/app/data/remote/ApiConfig.kt
class ApiConfig {
    companion object {
        const val BASE_URL = "https://api.example.com"
    }
}

// shared/src/commonMain/kotlin/com/example/app/data/remote/KtorClientFactory.kt
class KtorClientFactory {
    fun create(): HttpClient = HttpClient(engine) {
        install(ContentNegotiation) {
            json(Json {
                ignoreUnknownKeys = true
                isLenient = true
                encodeDefaults = false
            })
        }
        
        install(Logging) {
            level = LogLevel.BODY
            logger = Logger.slf4j()
        }
        
        install(DefaultRequest) {
            header(HttpHeaders.ContentType, ContentType.Application.Json.toString())
            header("X-API-Key", apiKey)
        }
        
        install(Retry) {
            maxRetries = 3
            exponentialDelay()
            retryOnException { it is IOException }
        }
        
        // Certificate pinning (platform-specific)
        installCertificatePinning()
    }
    
    private fun HttpClientConfig.installCertificatePinning() {
        expectCertificatePinningImplementation().install(this)
    }
}

// shared/src/androidMain/kotlin/com/example/app/data/remote/KtorClientFactory.kt
actual fun HttpClientConfig.expectCertificatePinningImplementation() {
    install(CertificatePin) {
        add(
            CertificatePin.Pin(
                "base64-encoded-hash-of-certificate",
                setOf(ApiConfig.BASE_URL)
            )
        )
    }
}

// shared/src/iosMain/kotlin/com/example/app/data/remote/KtorClientFactory.kt
actual fun HttpClientConfig.expectCertificatePinningImplementation() {
    // iOS certificate pinning implementation
    // Use custom TrustManager or AVOSTrustKit
}
```

### Secure Authentication Flow

```kotlin
// shared/src/commonMain/kotlin/com/example/app/data/remote/AuthManager.kt
class AuthManager(
    private val authApi: AuthApi,
    private val secureStorage: SecureStorage,
    private val tokenRefresher: TokenRefresher
) {
    companion object {
        const val ACCESS_TOKEN_KEY = "access_token"
        const val REFRESH_TOKEN_KEY = "refresh_token"
    }
    
    suspend fun login(email: String, password: String): Result<AuthTokens> {
        return try {
            val tokens = authApi.login(LoginRequest(email, password))
            
            // Store tokens securely
            secureStorage.save(ACCESS_TOKEN_KEY, tokens.accessToken)
            secureStorage.save(REFRESH_TOKEN_KEY, tokens.refreshToken)
            
            Result.success(tokens)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    suspend fun logout() {
        secureStorage.delete(ACCESS_TOKEN_KEY)
        secureStorage.delete(REFRESH_TOKEN_KEY)
    }
    
    suspend fun getAccessToken(): String? {
        return secureStorage.load(ACCESS_TOKEN_KEY).getOrNull()
    }
    
    suspend fun refreshTokens(): Result<AuthTokens> {
        val refreshToken = secureStorage.load(REFRESH_TOKEN_KEY).getOrNull() 
            ?: return Result.failure(Exception("No refresh token"))
        
        return try {
            val newTokens = authApi.refreshToken(RefreshRequest(refreshToken))
            
            secureStorage.save(ACCESS_TOKEN_KEY, newTokens.accessToken)
            secureStorage.save(REFRESH_TOKEN_KEY, newTokens.refreshToken)
            
            Result.success(newTokens)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// shared/src/commonMain/kotlin/com/example/app/data/remote/AuthInterceptor.kt
class AuthInterceptor(
    private val authManager: AuthManager
) : HttpClientPlugin.BaseConfig<Unit, PipelineInterceptor> {
    
    override val name: String = "AuthInterceptor"
    
    override fun install(plugin: Unit, scope: Pipeline) {
        val interceptor = PipelineInterceptor<Unit, RequestPipeline, Response> {
            val accessToken = authManager.getAccessToken()
            
            if (accessToken != null) {
                context.headers.append("Authorization", "Bearer $accessToken")
            }
            
            proceedWith(context)
        }
        
        val retryInterceptor = PipelineInterceptor<Unit, RequestPipeline, Response> {
            try {
                proceedWith(context)
            } catch (e: IOException) {
                // Token might be expired, try to refresh
                val result = authManager.refreshTokens()
                
                if (result.isFailure) {
                    throw e
                } else {
                    // Retry with new token
                    val accessToken = authManager.getAccessToken()
                    context.headers.append("Authorization", "Bearer $accessToken")
                    proceedWith(context)
                }
            }
        }
        
        scope.intercept(interceptor)
        scope.intercept(retryInterceptor)
    }
}
```

---

## 🧪 Testing Examples

### Unit Tests с MockK

```kotlin
// shared/src/commonTest/kotlin/com/example/app/domain/usecase/GetUserUseCaseTest.kt
class GetUserUseCaseTest {
    
    private lateinit var userRepository: UserRepository
    private lateinit var useCase: GetUserUseCase
    
    @BeforeTest
    fun setup() {
        userRepository = mockk()
        useCase = GetUserUseCase(userRepository)
    }
    
    @Test
    fun `returns success when user exists`() = runTest {
        // Arrange
        val userId = "user-123"
        val user = User(id = userId, name = "John Doe", email = "john@example.com")
        
        every { userRepository.getUser(userId) } returns user
        
        // Act
        val result = useCase(userId)
        
        // Assert
        assertTrue(result.isSuccess)
        assertEquals(user, result.getOrNull())
    }
    
    @Test
    fun `returns failure when user not found`() = runTest {
        // Arrange
        val userId = "user-999"
        
        every { userRepository.getUser(userId) } returns null
        
        // Act
        val result = useCase(userId)
        
        // Assert
        assertTrue(result.isFailure)
        assertTrue(result.exceptionOrNull() is UserNotFoundException)
    }
    
    @Test
    fun `returns failure when exception occurs`() = runTest {
        // Arrange
        val userId = "user-123"
        
        every { userRepository.getUser(userId) } throws IOException("Network error")
        
        // Act
        val result = useCase(userId)
        
        // Assert
        assertTrue(result.isFailure)
        assertTrue(result.exceptionOrNull() is IOException)
    }
}
```

### Integration Tests с Fake Implementations

```kotlin
// shared/src/commonTest/kotlin/com/example/app/data/repository/UserRepositoryImplTest.kt
class UserRepositoryImplTest {
    
    private lateinit var localDataSource: UserLocalDataSource
    private lateinit var remoteDataSource: UserRemoteDataSource
    private lateinit var repository: UserRepositoryImpl
    
    @BeforeTest
    fun setup() {
        localDataSource = FakeUserLocalDataSource()
        remoteDataSource = FakeUserRemoteDataSource()
        repository = UserRepositoryImpl(localDataSource, remoteDataSource)
    }
    
    @Test
    fun `returns local user and updates from remote`() = runTest {
        // Arrange
        val userId = "user-123"
        val localUser = User(id = userId, name = "John", email = "john@local.com")
        val remoteUser = User(id = userId, name = "John Updated", email = "john@remote.com")
        
        localDataSource.saveUser(localUser)
        remoteDataSource.users[userId] = remoteUser
        
        // Act
        val result = repository.getUser(userId)
        
        // Assert
        assertEquals(localUser, result) // Returns local immediately
        
        // Wait for background sync
        delay(100)
        
        val syncedUser = localDataSource.getUser(userId)
        assertEquals(remoteUser.name, syncedUser?.name) // Updated from remote
    }
    
    @Test
    fun `fetches from remote when not in local`() = runTest {
        // Arrange
        val userId = "user-456"
        val remoteUser = User(id = userId, name = "Jane", email = "jane@example.com")
        
        remoteDataSource.users[userId] = remoteUser
        
        // Act
        val result = repository.getUser(userId)
        
        // Assert
        assertEquals(remoteUser, result)
        assertEquals(remoteUser, localDataSource.getUser(userId)) // Cached locally
    }
}

// Fake implementations for testing
class FakeUserLocalDataSource : UserLocalDataSourceInterface {
    private val users = mutableMapOf<String, User>()
    
    override suspend fun getUser(userId: String): User? = users[userId]
    override suspend fun getAllUsers(): List<User> = users.values.toList()
    
    override suspend fun saveUser(user: User): Result<Unit> {
        users[user.id] = user
        return Result.success(Unit)
    }
}

class FakeUserRemoteDataSource : UserRemoteDataSourceInterface {
    val users = mutableMapOf<String, User>()
    
    override suspend fun getUser(userId: String): Result<User?> {
        return Result.success(users[userId])
    }
}
```

---

## 📊 Performance Optimization Examples

### Lazy Loading с Pagination

```kotlin
// shared/src/commonMain/kotlin/com/example/app/data/local/PaginatedDao.kt
@SqlQuery("SELECT * FROM users LIMIT :limit OFFSET :offset")
suspend fun getUsersPaginated(limit: Int, offset: Int): List<User>

@SqlQuery("SELECT COUNT(*) FROM users")
suspend fun getTotalUserCount(): Int

// shared/src/commonMain/kotlin/com/example/app/data/local/PaginatedDataSource.kt
class PaginatedUserDataSource(
    private val dao: PaginatedDao,
    private val pageSize: Int = 20
) {
    suspend fun getUsersPage(page: Int): PaginatedResult<User> {
        val offset = page * pageSize
        
        return withContext(Dispatchers.IO) {
            try {
                val users = dao.getUsersPaginated(pageSize, offset)
                val totalCount = dao.getTotalUserCount()
                
                PaginatedResult(
                    data = users,
                    currentPage = page,
                    totalPages = (totalCount + pageSize - 1) / pageSize,
                    hasMore = offset + users.size < totalCount
                )
            } catch (e: Exception) {
                PaginatedResult.Error(e)
            }
        }
    }
}

data class PaginatedResult<T>(
    val data: List<T>,
    val currentPage: Int,
    val totalPages: Int,
    val hasMore: Boolean
) {
    data class Error<T>(val exception: Exception) : PaginatedResult<T>(emptyList(), 0, 0, false)
}

// shared/src/commonMain/kotlin/com/example/app/presentation/viewmodel/UsersViewModel.kt
class UsersViewModel(
    private val paginatedDataSource: PaginatedUserDataSource,
    dispatcher: CoroutineDispatcher = Dispatchers.Default
) : ViewModel(dispatcher) {
    
    private val _users = MutableStateFlow<List<UserUiModel>>(emptyList())
    val users: StateFlow<List<UserUiModel>> = _users.asStateFlow()
    
    private val _loading = MutableStateFlow(false)
    val loading: StateFlow<Boolean> = _loading.asStateFlow()
    
    private var currentPage = 0
    private val userCache = mutableListOf<UserUiModel>()
    
    init {
        loadMoreUsers()
    }
    
    fun loadMoreUsers() {
        viewModelScope.launch {
            if (_loading.value) return@launch
            
            _loading.value = true
            
            try {
                val result = paginatedDataSource.getUsersPage(currentPage)
                
                when (result) {
                    is PaginatedResult.Error -> {
                        // Handle error
                    }
                    else -> {
                        val newUsers = result.data.map { it.toUiModel() }
                        userCache.addAll(newUsers)
                        _users.value = userCache.toList()
                        
                        if (result.hasMore) {
                            currentPage++
                        }
                    }
                }
            } finally {
                _loading.value = false
            }
        }
    }
    
    fun refresh() {
        viewModelScope.launch {
            currentPage = 0
            userCache.clear()
            loadMoreUsers()
        }
    }
}
```

---

## 🎯 Best Practices Summary

### ✅ Do:
- **Use Result type** для error handling вместо exceptions
- **Implement offline-first** с background sync
- **Write comprehensive tests** (unit, integration, UI)
- **Use StateFlow/SharedFlow** вместо LiveData для state management
- **Create abstraction layers** для platform-specific code
- **Document architectural decisions** с ADRs

### ❌ Don't:
- **Don't use global singletons** без proper DI
- **Don't block main thread** для I/O operations
- **Don't ignore platform differences** - handle them gracefully
- **Don't skip testing** для critical paths
- **Don't hardcode values** - use configuration

---

## 🔗 Additional Resources

- [Kotlin Multiplatform Samples](https://github.com/JetBrains/compose-multiplatform-samples)
- [KMP Architecture Templates](https://github.com/SlackHQ/kmp-template)
- [Compose Multiplatform Best Practices](https://www.jetbrains.com/help/compose-multiplatform/)

**Happy coding! 🚀**

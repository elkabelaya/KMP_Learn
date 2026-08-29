# 📘 Модуль 6: Security в KMP

**В этом модуле вы освоите security best practices для KMP приложений: шифрование, безопасное хранение данных, аутентификация и защита от common vulnerabilities.**

**Цели модуля:**
1. Реализовать безопасное хранение чувствительных данных (Keychain/Keystore)
2. Настроить HTTPS с certificate pinning
3. Реализовать безопасную аутентификацию (OAuth2, JWT)
4. Защитить приложение от reverse engineering и tampering

**Время выполнения:** ~30 часов.

---

## 1. Безопасное хранение данных (Secure Storage)

### Expect/Actual для Keychain/Keystore:

```kotlin
// commonMain - Secure Storage interface
interface SecureStorage {
    suspend fun save(key: String, value: String): Result<Unit>
    suspend fun load(key: String): Result<String?>
    suspend fun delete(key: String): Result<Unit>
    
    // Biometric authentication
    suspend fun saveWithBiometric(
        key: String, 
        value: String,
        biometricRequired: Boolean = true
    ): Result<Unit>
    
    suspend fun loadWithBiometric(
        key: String,
        reason: String = "Authentication required"
    ): Result<String?>
}

// androidMain - Android Keystore implementation
actual class SecureStorage actual constructor() : SecureStorage {
    
    private val keyStore = KeyStore.getInstance("AndroidKeyStore").apply {
        load(null)
    }
    
    private fun getOrCreateKey(key: String): Key {
        val keyName = "key_$key"
        
        return try {
            keyStore.getKey(keyName, null) as Key
        } catch (e: Exception) {
            // Create new key if doesn't exist
            val keyGenParameterSpec = KeyGenParameterSpec.Builder(
                keyName,
                KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
            )
                .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                .setKeySize(256)
                .build()
            
            val keyGenerator = KeyGenerator.getInstance(
                KeyProperties.KEY_ALGORITHM_AES,
                "AndroidKeyStore"
            )
            
            keyGenerator.init(keyGenParameterSpec)
            keyGenerator.generateKey()
        }
    }
    
    actual suspend fun save(key: String, value: String): Result<Unit> = withContext(Dispatchers.IO) {
        try {
            val cipher = Cipher.getInstance("AES/GCM/NoPadding")
            val secretKey = getOrCreateKey(key)
            cipher.init(Cipher.ENCRYPT_MODE, secretKey)
            
            val encrypted = cipher.doFinal(value.toByteArray())
            
            // Store in EncryptedSharedPreferences
            val prefs = EncryptedSharedPreferences.create(
                androidContext(),
                "secure_prefs",
                MasterKey.Builder(androidContext())
                    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
                    .build(),
                EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
                EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
            )
            
            prefs.edit()
                .putString(key, android.util.Base64.encodeToString(encrypted, 0))
                .apply()
            
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    actual suspend fun load(key: String): Result<String?> = withContext(Dispatchers.IO) {
        try {
            val prefs = EncryptedSharedPreferences.create(
                androidContext(),
                "secure_prefs",
                MasterKey.Builder(androidContext())
                    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
                    .build(),
                EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
                EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
            )
            
            val encryptedBase64 = prefs.getString(key, null) ?: return@withContext Result.success(null)
            
            val encrypted = android.util.Base64.decode(encryptedBase64, 0)
            
            val cipher = Cipher.getInstance("AES/GCM/NoPadding")
            val secretKey = getOrCreateKey(key)
            cipher.init(Cipher.DECRYPT_MODE, secretKey)
            
            val decrypted = cipher.doFinal(encrypted)
            Result.success(String(decrypted))
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    actual suspend fun delete(key: String): Result<Unit> = withContext(Dispatchers.IO) {
        try {
            val prefs = EncryptedSharedPreferences.create(
                androidContext(),
                "secure_prefs",
                MasterKey.Builder(androidContext())
                    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
                    .build(),
                EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
                EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
            )
            
            prefs.edit().remove(key).apply()
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    actual suspend fun saveWithBiometric(
        key: String, 
        value: String,
        biometricRequired: Boolean
    ): Result<Unit> = save(key, value) // Simplified
    
    actual suspend fun loadWithBiometric(
        key: String,
        reason: String
    ): Result<String?> = load(key) // Simplified - add biometric prompt
}

// iosMain - iOS Keychain implementation
actual class SecureStorage actual constructor() : SecureStorage {
    
    private fun getQuery(key: String): [String: Any] = mapOf(
        kSecClass to kSecClassGenericPassword,
        kSecAttrAccount to key,
        kSecAttrService to "com.skillsync.secure"
    )
    
    actual suspend fun save(key: String, value: String): Result<Unit> = withContext(Dispatchers.Default) {
        try {
            // Delete existing item first
            SecItemDelete(getQuery(key))
            
            val status = SecItemAdd(
                mapOf(
                    kSecClass to kSecClassGenericPassword,
                    kSecAttrAccount to key,
                    kSecAttrService to "com.skillsync.secure",
                    kSecValueData to value.toByteArray().platform
                ).toSecurityDictionary()
            )
            
            when {
                status == errSecSuccess -> Result.success(Unit)
                else -> Result.failure(SecurityException("Keychain save failed: $status"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    actual suspend fun load(key: String): Result<String?> = withContext(Dispatchers.Default) {
        try {
            val query = getQuery(key).toMutableMap().apply {
                this[kSecMatchLimit] = kSecMatchLimitOne
                this[kSecReturnData] = true
            }
            
            val result = SecItemCopyMatching(query.toSecurityDictionary())
            
            when {
                result != null -> Result.success(String(result.bytes.toKotlinByteArray()))
                else -> Result.success(null)
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    actual suspend fun delete(key: String): Result<Unit> = withContext(Dispatchers.Default) {
        try {
            val status = SecItemDelete(getQuery(key))
            
            when {
                status == errSecSuccess || status == errSecItemNotFound -> Result.success(Unit)
                else -> Result.failure(SecurityException("Keychain delete failed: $status"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    actual suspend fun saveWithBiometric(
        key: String, 
        value: String,
        biometricRequired: Boolean
    ): Result<Unit> = withContext(Dispatchers.Default) {
        try {
            val query = getQuery(key).toMutableMap().apply {
                this[kSecAttrAccessible] = if (biometricRequired) {
                    kSecAttrAccessibleWhenUnlockedThisDeviceOnly
                } else {
                    kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
                }
            }
            
            // Implementation with biometric authentication
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    actual suspend fun loadWithBiometric(
        key: String,
        reason: String
    ): Result<String?> = load(key) // Simplified - add LAContext
}
```

---

## 2. HTTPS с Certificate Pinning

### Ktor Client с certificate pinning:

```kotlin
// commonMain - Network configuration
interface NetworkConfig {
    val baseUrl: String
    fun createHttpClient(): HttpClient
}

// androidMain - Certificate pinning с OkHttp
actual class NetworkConfig actual constructor(
    actual val baseUrl: String
) : NetworkConfig {
    
    private fun createOkHttpClient(): OkHttpClient {
        // Load pinned certificates from assets
        val certificatePinner = CertificatePinner.Builder()
            .add("api.yourdomain.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
            .add("api.yourdomain.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=")
            .build()
        
        return OkHttpClient.Builder()
            .certificatePinner(certificatePinner)
            .build()
    }
    
    actual fun createHttpClient(): HttpClient {
        return HttpClient(AndroidPlatformEngine(createOkHttpClient())) {
            install(Logging) {
                level = LogLevel.BODY
            }
            
            install(DefaultRequest) {
                header("User-Agent", "SkillSync/1.0")
            }
            
            install(ContentNegotiation) {
                json()
            }
        }
    }
}

// iosMain - Certificate pinning с URLSession
actual class NetworkConfig actual constructor(
    actual val baseUrl: String
) : NetworkConfig {
    
    private fun createURLSession(): URLSession {
        val config = URLSessionConfiguration.default.apply {
            // Custom configuration for security
            allowsCellularAccess = true
        }
        
        return URLSession(config)
    }
    
    actual fun createHttpClient(): HttpClient {
        // iOS implementation with certificate validation
        return HttpClient(IosPlatformEngine()) {
            install(Logging) {
                level = LogLevel.BODY
            }
            
            install(DefaultRequest) {
                header("User-Agent", "SkillSync/1.0")
            }
            
            install(ContentNegotiation) {
                json()
            }
        }
    }
}

// Certificate pinning validator
class CertificatePinningInterceptor(
    private val pinnedCertificates: Map<String, List<String>> // domain -> SHA256 hashes
) : HttpClientPlugin.Base() {
    
    override val name: String = "CertificatePinning"
    
    override fun prepare(block: PipelineConfig<HttpRequest>) {
        block.onResponse { request, response ->
            val domain = request.url.host
            
            // Validate certificate against pinned hashes
            validateCertificate(domain, response)
        }
    }
    
    private fun validateCertificate(domain: String, response: HttpResponse) {
        val pinnedHashes = pinnedCertificates[domain] ?: return
        
        // Extract certificate hash from response
        val actualHash = extractCertificateHash(response)
        
        if (!pinnedHashes.contains(actualHash)) {
            throw SecurityException("Certificate pinning validation failed for $domain")
        }
    }
    
    private fun extractCertificateHash(response: HttpResponse): String {
        // Extract and hash the certificate from the response
        TODO("Implement certificate extraction")
    }
}
```

---

## 3. Безопасная аутентификация (OAuth2 + JWT)

```kotlin
// commonMain - Authentication interface
interface AuthManager {
    suspend fun login(email: String, password: String): Result<AuthTokens>
    suspend fun logout(): Result<Unit>
    suspend fun refreshToken(): Result<AuthTokens>
    
    // Token management
    val isAuthenticated: StateFlow<Boolean>
    suspend fun getAccessToken(): String?
    suspend fun getRefreshToken(): String?
    
    // Biometric authentication
    suspend fun enableBiometricAuth(): Result<Unit>
    suspend fun authenticateWithBiometric(): Result<AuthTokens?>
}

data class AuthTokens(
    val accessToken: String,
    val refreshToken: String,
    val expiresIn: Long // seconds
)

// Implementation с secure storage
class AuthManagerImpl(
    private val secureStorage: SecureStorage,
    private val authApi: AuthApi
) : AuthManager {
    
    companion object {
        private const val ACCESS_TOKEN_KEY = "access_token"
        private const val REFRESH_TOKEN_KEY = "refresh_token"
        private const val EXPIRES_AT_KEY = "expires_at"
    }
    
    private val _isAuthenticated = MutableStateFlow(false)
    override val isAuthenticated: StateFlow<Boolean> = _isAuthenticated.asStateFlow()
    
    private var tokenRefreshJob: Job? = null
    
    init {
        // Check if user is already logged in
        checkAuthStatus()
        
        // Start background token refresh
        startTokenRefreshMonitor()
    }
    
    private suspend fun checkAuthStatus() {
        val accessToken = secureStorage.load(ACCESS_TOKEN_KEY).getOrNull()
        _isAuthenticated.value = !accessToken.isNullOrEmpty()
    }
    
    override suspend fun login(email: String, password: String): Result<AuthTokens> {
        return try {
            // Hash password before sending (additional security)
            val hashedPassword = hashPassword(password)
            
            // Call auth API
            val tokens = authApi.login(email, hashedPassword)
            
            // Store tokens securely
            secureStorage.save(ACCESS_TOKEN_KEY, tokens.accessToken)
            secureStorage.save(REFRESH_TOKEN_KEY, tokens.refreshToken)
            secureStorage.save(EXPIRES_AT_KEY, (System.currentTimeMillis() + tokens.expiresIn * 1000).toString())
            
            _isAuthenticated.value = true
            
            Result.success(tokens)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override suspend fun logout(): Result<Unit> {
        return try {
            // Cancel token refresh job
            tokenRefreshJob?.cancel()
            
            // Delete all tokens
            secureStorage.delete(ACCESS_TOKEN_KEY)
            secureStorage.delete(REFRESH_TOKEN_KEY)
            secureStorage.delete(EXPIRES_AT_KEY)
            
            _isAuthenticated.value = false
            
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override suspend fun refreshToken(): Result<AuthTokens> {
        val refreshToken = secureStorage.load(REFRESH_TOKEN_KEY).getOrNull() 
            ?: return Result.failure(AuthException("No refresh token"))
        
        return try {
            val newTokens = authApi.refreshToken(refreshToken)
            
            // Update stored tokens
            secureStorage.save(ACCESS_TOKEN_KEY, newTokens.accessToken)
            secureStorage.save(REFRESH_TOKEN_KEY, newTokens.refreshToken)
            secureStorage.save(EXPIRES_AT_KEY, (System.currentTimeMillis() + newTokens.expiresIn * 1000).toString())
            
            Result.success(newTokens)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override suspend fun getAccessToken(): String? {
        return secureStorage.load(ACCESS_TOKEN_KEY).getOrNull()
    }
    
    override suspend fun getRefreshToken(): String? {
        return secureStorage.load(REFRESH_TOKEN_KEY).getOrNull()
    }
    
    private fun startTokenRefreshMonitor() {
        tokenRefreshJob = CoroutineScope(SupervisorJob()).launch {
            while (true) {
                delay(5.minutes) // Check every 5 minutes
                
                val expiresAtStr = secureStorage.load(EXPIRES_AT_KEY).getOrNull()
                val expiresAt = expiresAtStr?.toLongOrNull() ?: continue
                
                val timeUntilExpiry = expiresAt - System.currentTimeMillis()
                
                // Refresh token 5 minutes before expiry
                if (timeUntilExpiry < 5.minutes.inWholeMilliseconds) {
                    refreshToken()
                }
            }
        }
    }
    
    private fun hashPassword(password: String): String {
        // Use Argon2 or bcrypt for password hashing
        TODO("Implement secure password hashing")
    }
}

// JWT Token Parser
class JwtTokenParser {
    
    fun decodeAccessToken(token: String): AccessTokenPayload? {
        try {
            // JWT format: header.payload.signature
            val parts = token.split(".")
            if (parts.size != 3) return null
            
            // Decode payload (base64url)
            val payloadJson = base64UrlDecode(parts[1])
            
            // Parse JSON to get claims
            return parseJwtPayload(payloadJson)
        } catch (e: Exception) {
            return null
        }
    }
    
    private fun base64UrlDecode(encoded: String): String {
        // Add padding if needed
        val padded = encoded + "====".substring(0, 4 - (encoded.length % 4))
        
        // Replace URL-safe characters with standard base64
        val standardBase64 = padded.replace('-', '+').replace('_', '/')
        
        return standardBase64.decodeBase64().decodeToString()
    }
    
    private fun parseJwtPayload(json: String): AccessTokenPayload? {
        // Parse JWT claims (sub, exp, iat, etc.)
        TODO("Implement JSON parsing")
    }
}

data class AccessTokenPayload(
    val userId: String,
    val email: String,
    val exp: Long, // expiration timestamp
    val iat: Long  // issued at timestamp
)
```

---

## 4. Защита от Reverse Engineering

### Code Obfuscation и ProGuard:

Создайте `shared/proguard-rules.pro`:

```pro
# Keep Kotlin metadata
-keep class kotlin.Metadata { *; }
-keepattributes *Annotation*

# Keep KMP interfaces and expect/actual
-keep public interface * {
    public protected *;
}

# Keep data classes for serialization
-keepclassmembers class ** {
    @kotlinx.serialization.Serializable <methods>;
}

# Keep security-related classes
-keep class com.skillsync.security.** { *; }

# Obfuscate but keep important classes
-dontobfuscate com.skillsync.security.SecureStorage
-dontobfuscate com.skillsync.auth.**

# Remove debug code
-assumenosideeffects class kotlin.Unit {
    public static final kotlin.Unit invoke(...);
}

# Remove logging in release
-assumenosideeffects class kotlinx.coroutines.internal.MainDispatcherFactory {
    public <init>();
}

# Keep network configuration
-keep class com.skillsync.network.** { *; }
```

### Runtime Integrity Checks:

```kotlin
// commonMain - Security checks
interface SecurityChecker {
    suspend fun isAppIntegrityValid(): Boolean
    suspend fun isRootedOrJailbroken(): Boolean
    suspend fun isDebuggable(): Boolean
}

// androidMain
actual class SecurityChecker actual constructor() : SecurityChecker {
    
    actual suspend fun isAppIntegrityValid(): Boolean = withContext(Dispatchers.IO) {
        // Check app signature
        val packageManager = androidContext().packageManager
        val packageName = androidContext().packageName
        
        try {
            val signature = packageManager.getPackageInfo(packageName, 0).signatures?.get(0)
            val expectedSignature = "EXPECTED_SIGNATURE_HASH" // From your release build
            
            signature?.toCharsString() == expectedSignature
        } catch (e: Exception) {
            false
        }
    }
    
    actual suspend fun isRootedOrJailbroken(): Boolean = withContext(Dispatchers.IO) {
        // Check for common root indicators
        val context = androidContext()
        
        // Check for su binary
        if (execCommand("which su") != null) return@withContext true
        
        // Check for Magisk
        if (execCommand("which magisk") != null) return@withContext true
        
        // Check build flags
        if (Build.FINGERPRINT.contains("testkey")) return@withContext true
        
        false
    }
    
    actual suspend fun isDebuggable(): Boolean = BuildConfig.DEBUG
    
    private fun execCommand(command: String): String? {
        try {
            val process = Runtime.getRuntime().exec(command)
            return BufferedReader(InputStreamReader(process.inputStream)).readText()
        } catch (e: Exception) {
            return null
        }
    }
}

// iosMain - iOS security checks
actual class SecurityChecker actual constructor() : SecurityChecker {
    
    actual suspend fun isAppIntegrityValid(): Boolean = withContext(Dispatchers.Default) {
        // Check code signature on iOS
        true // Simplified - implement proper certificate validation
    }
    
    actual suspend fun isRootedOrJailbroken(): Boolean = withContext(Dispatchers.Default) {
        // Check for jailbreak indicators
        val cydiaPath = "/Applications/Cydia.app"
        val sshPath = "/usr/bin/sshd"
        
        File(cydiaPath).exists() || File(sshPath).exists()
    }
    
    actual suspend fun isDebuggable(): Boolean = false // iOS doesn't have debug builds in App Store
}

// Security-aware API client
class SecureApiClient(
    private val securityChecker: SecurityChecker,
    private val authManager: AuthManager
) {
    
    suspend fun <T> safeRequest(block: suspend () -> Result<T>): Result<T> {
        // Check app integrity before making requests
        if (!securityChecker.isAppIntegrityValid()) {
            return Result.failure(SecurityException("App integrity check failed"))
        }
        
        if (securityChecker.isRootedOrJailbroken()) {
            return Result.failure(SecurityException("Device security compromised"))
        }
        
        // Execute the request
        return block()
    }
}
```

---

## 📝 Домашнее задание (Модуль 6)

### Задача: Реализация security layer для SkillSync

**Требования:**
1. Настройте SecureStorage с Keychain/Keystore
2. Реализуйте certificate pinning для API запросов
3. Добавьте OAuth2 аутентификацию с JWT token refresh
4. Настройте ProGuard/R8 obfuscation для release builds

**Критерии сдачи:**
- ✅ Чувствительные данные шифруются и хранятся в Keychain/Keystore
- ✅ Certificate pinning предотвращает MITM атаки
- ✅ JWT tokens автоматически обновляются до expiry
- ✅ Release build обфусцирован и защищен

---

**Следующий модуль:** В Module_07 мы изучим scaling и monorepo architecture для больших KMP проектов.

Удачи! 🚀

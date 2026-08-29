# 🔒 Security Guide

**Security best practices для KMP applications - protect your users и data**

---

## 📋 Table о Contents
- [Security Principles](#security-principles)  
- [Authentication и Authorization](#authentication-and-authorization)
- [Data Security](#data-security)  
- [Network Security](#network-security)
- [Platform-Specific Security](#platform-specific-security)  
- [Security Testing](#security-testing)

---

## Security Principles

### 1. Defense in Depth
**Never rely on a single security measure.** Layer multiple protections:

```kotlin
// Example: Multiple layers о protection  
class SecureApiCall {
    // Layer 1: Input validation
    suspend fun fetchData(userId: String): Result<Data> {
        require(validUserId(userId)) { "Invalid user ID" }
        
        // Layer 2: Authentication  
        require(authenticated()) { "Not authenticated" }
        
        // Layer 3: Authorization  
        require(authorized(userId)) { "Not authorized" }
        
        // Layer 4: Secure network call  
        return secureHttpClient.get("/users/$userId")
    }
}
```

### 2. Least Privilege  
**Grant minimum permissions needed:**

```kotlin
// ❌ Bad: Requesting all permissions  
@AndroidPermission(permissions = [
    Manifest.permission.CAMERA,
    Manifest.permission.LOCATION,  
    Manifest.permission.CONTACTS
])

// ✅ Good: Request only what's needed  
@AndroidPermission(permissions = [Manifest.permission.CAMERA])
```

### 3. Fail Securely  
**When in doubt, deny access:**

```kotlin
// ❌ Bad: Default к allowing  
fun canAccess(resource: String): Boolean {
    return checkPermissions() ?: true  // Defaults к true on error!
}

// ✅ Good: Default к denying  
fun canAccess(resource: String): Boolean {
    return checkPermissions() ?: false  // Defaults к false on error
}
```

---

## Authentication и Authorization

### Secure Token Storage

#### Android:
```kotlin
// ✅ Use EncryptedSharedPreferences  
val encryptedPrefs = EncryptedSharedPreferences.create(
    context,
    "secure_prefs",
    MasterKey.Builder(context).build(),
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)

encryptedPrefs.edit()
    .putString("auth_token", token)
    .apply()

// ❌ Don't use regular SharedPreferences for tokens!  
```

#### iOS:
```swift
// ✅ Use Keychain  
func saveToken(_ token: String) throws {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: "auth_token",
        kSecValueData as String: token.data(using: .utf8)!
    ]
    
    SecItemDelete(query as CFDictionary)  // Remove existing  
    SecItemAdd(query as CFDictionary, nil)
}

// ❌ Don't use UserDefaults for tokens!  
```

#### KMP Shared (Recommended):
```kotlin
// Use SecureStore library  
class AuthRepository(
    private val secureStore: SecureStore
) {
    suspend fun saveToken(token: String): Result<Unit> {
        return try {
            secureStore.set("auth_token", token)
            Result.Success(Unit)
        } catch (e: Exception) {
            Result.Failure(e)
        }
    }
    
    suspend fun getToken(): Result<String?> {
        return try {
            val token = secureStore.get("auth_token")
            Result.Success(token)
        } catch (e: Exception) {
            Result.Failure(e)
        }
    }
}
```

### Token Refresh Strategy

```kotlin
class AuthManager(
    private val tokenStorage: TokenStorage,  
    private val apiClient: ApiClient
) {
    private var isRefreshing = false
    private val refreshMutex = Mutex()
    
    suspend fun getValidToken(): Result<String> {
        val token = tokenStorage.getToken()
        
        // If token is valid, return it  
        if (token != null && !isTokenExpired(token)) {
            return Result.Success(token)
        }
        
        // Refresh token  
        return refreshMutex.withLock {
            if (isRefreshing) {
                // Wait for another refresh к complete  
                return waitForRefresh()
            }
            
            isRefreshing = true
            try {
                val newToken = apiClient.refreshToken()
                tokenStorage.saveToken(newToken)
                Result.Success(newToken)
            } finally {
                isRefreshing = false
            }
        }
    }
    
    private fun isTokenExpired(token: String): Boolean {
        val expiry = decodeTokenExpiry(token)
        return System.currentTimeMillis() > expiry
    }
}
```

### Biometric Authentication

#### Android:
```kotlin
class BiometricAuth(
    private val context: Context
) {
    suspend fun authenticate(): Result<Unit> {
        return trySend(
            biometricPrompt = BiometricPrompt(
                context,
                callback = object : BiometricPrompt.AuthenticationCallback() {
                    override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                        // Grant access  
                    }
                    
                    override fun onAuthenticationError(error: Int, errString: CharSequence) {
                        // Deny access  
                    }
                },
                authCallback = object : BiometricPrompt.AuthenticationCallback() {
                    override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                        // Grant access  
                    }
                }
            ),
            title = "Authenticate",
            subtitle = "Use your fingerprint к access secure data"
        )
    }
}
```

#### iOS:
```swift
class BiometricAuth {
    func authenticate() async throws {
        let context = LAContext()
        var error: NSError?
        
        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
            throw BiometricError.notAvailable  
        }
        
        try await context.evaluatePolicy(
            .deviceOwnerAuthenticationWithBiometrics,
            localizedReason: "Use your biometric data к access secure information"
        )
    }
}
```

---

## Data Security

### Encryption at Rest

#### Database Encryption:
```kotlin
// SQLDelite with encryption  
class EncryptedDatabase(
    context: Context  // Android
) {
    val database = SQLiteDatabase.build {
        name("secure.db")
        
        // Enable encryption  
        setKey(deriveEncryptionKey())
    }
}

// Derive key from user password  
fun deriveEncryptionKey(): ByteArray {
    val salt = generateSalt()  // Store this securely  
    val key = PBKDF2Hasher.hash(
        password = userPassword,
        salt = salt,  
        iterations = 100000
    )
    return key
}
```

### Sensitive Data Handling

#### Never Log Sensitive Information:
```kotlin
// ❌ Bad: Logging sensitive data  
Log.d("USER", "User data: $userData")  // May contain PII!

// ✅ Good: Log only safe information  
Log.d("USER", "User data loaded successfully")
```

#### Sanitize Error Messages:
```kotlin
// ❌ Bad: Exposing internal details  
throw Exception("Database query failed: SELECT * FROM users WHERE id = $id")

// ✅ Good: Generic error message  
throw Exception("Failed к load user data. Please try again.")
```

### Data Minimization

```kotlin
// ❌ Bad: Collecting unnecessary data  
data class User(
    val id: String,
    val name: String,  
    val email: String,
    val phone: String,      // Do you need this?
    val address: String,    // Do you need this?  
    val birthDate: String,  // Do you need this?
    val ssn: String        // NEVER collect this!
)

// ✅ Good: Only collect what's necessary  
data class User(
    val id: String,
    val name: String,
    val email: String      // Only what you need
)
```

---

## Network Security

### HTTPS Only

```kotlin
// Ktor client с HTTPS enforcement  
val httpClient = HttpClient(CIO) {
    install(SSL) {
        // Only allow HTTPS  
        connector = SSLConnectorBuilder()
            .configure { 
                // Configure certificate validation  
            }
    }
    
    install(Logging) {
        level = Logger.Level.INFO  
        // Never log sensitive headers!
    }
}

// ❌ Don't disable certificate validation in production!  
// installer(SSL) { 
//     validator = TrustAllTrustManager  // DANGEROUS!
// }
```

### Certificate Pinning

```kotlin
class SecureHttpClient {
    val client = HttpClient(CIO) {
        install(SSL) {
            certificatePinner = CertificatePinner { hostname ->
                when (hostname) {
                    "api.yourapp.com" -> listOf(
                        "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",  
                        "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB="  // Backup
                    )
                    else -> emptyList()
                }
            }
        }
    }
}
```

### Request Security Headers

```kotlin
// Add security headers к all requests  
class SecurityInterceptor : Interceptor {
    override suspend fun intercept(call: HttpRequestPipeline): HttpResponse {
        call.headers.append("X-API-Key", apiKey)  
        call.headers.append("X-Request-ID", generateUUID())
        
        return proceed(call)
    }
}

// Validate headers on server side  
fun validateRequest(request: HttpRequest): Boolean {
    return request.headers["X-API-Key"] != null &&
           request.headers["X-Request-ID"] != null
}
```

---

## Platform-Specific Security

### Android Security

#### 1. ProGuard/R8 Obfuscation
```kotlin
// proguard-rules.pro  
-keep public class * extends androidx.compose.runtime.Applier  
-keepclassmembers class * {
    @com.squareup.moshi.JsonAdapter <methods>;
}

// Keep your security classes  
-keep class com.yourapp.security.** { *; }
```

#### 2. Network Security Configuration  
```xml
<!-- res/xml/network_security_config.xml -->  
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>  
        </trust-anchors>
    </base-config>
    
    <domain-config domain="api.yourapp.com">
        <trust-anchors>
            <certificates src="@raw/api_certificate"/>  
        </trust-anchors>
    </domain-config>
</network-security-config>
```

#### 3. Keystore Integration  
```kotlin
// Store secrets в AndroidKeystore  
class KeyStoreManager(private val context: Context) {
    private val keyStore = KeyStore.getInstance("AndroidKeyStore").apply {
        load(null)
    }
    
    fun storeSecret(key: String, secret: String) {
        val secretKey = generateSecretKey(secret)
        keyStore.setSecretEntry(key, KeyStore.SecretKeyEntry(secretKey))
    }
}
```

### iOS Security

#### 1. Keychain Access Control  
```swift
// Use appropriate access control  
let accessControl = LAContext()
try accessControl.save(
    data: tokenData,  
    accessControl: .biometryCurrentSet  // Requires biometric auth
)
```

#### 2. App Transport Security  
```xml
<!-- Info.plist -->  
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>  
    <false/>  <!-- Never enable this in production! -->
    
    <key>NSExceptionDomains</key>  
    <dict>
        <key>api.yourapp.com</key>  
        <dict>
            <key>NSIncludesSubdomains</key>  
            <true/>
            <key>NSTemporaryExceptionAllowsInsecureHTTPLoads</key>  
            <false/>
        </dict>
    </dict>
</dict>
```

#### 3. Jailbreak Detection  
```swift
func isJailbroken() -> Bool {
    // Check for common jailbreak indicators  
    return FileManager.default.fileExists(atPath: "/Applications/Cydia.app") ||
           FileManager.default.fileExists(atPath: "/private/etc/apt/") ||
           system("which sshd") == 0
}

// Warn users or restrict functionality if jailbroken  
if isJailbroken() {
    showSecurityWarning()
}
```

---

## Security Testing

### 1. Static Analysis

#### Detekt (Kotlin):
```kotlin
// build.gradle.kts  
dependencies {
    detekt("io.gitlab.arturbosch.detekt:detekt-cli:1.23.0")
}

// Run security checks  
./gradlew detekt
```

#### Custom Security Rules:
```kotlin
// detekt-config.yml  
security:  
  PrintStackTrace:
    active: true  # Don't print stack traces
  
  SecureRandom:  
    active: true  # Use SecureRandom, not Random
```

### 2. Dynamic Analysis

#### OWASP ZAP:
```bash
# Scan your API endpoints  
zap-cli quick-scan https://api.yourapp.com

# Generate report  
zap-cli report -o security-report.html
```

### 3. Penetration Testing

**What к test:**
- Authentication bypass  
- Authorization flaws  
- Input validation  
- Data encryption  
- Session management

**Tools:**
- **Burp Suite:** Manual testing  
- **MobSF:** Mobile security framework  
- **OWASP ZAP:** Automated scanning

### 4. Security Checklist

**Before Release:**
- [ ] All API calls use HTTPS  
- [ ] Sensitive data encrypted at rest  
- [ ] No hardcoded secrets в code
- [ ] Certificate pinning implemented  
- [ ] Proper error handling (no info leakage)
- [ ] Input validation on all endpoints  
- [ ] Authentication required for sensitive operations  
- [ ] Session timeout implemented
- [ ] Security headers configured

---

## Incident Response

### If You Discover a Vulnerability:

#### 1. Don't Publicly Disclose
**Wait for us к fix it first!**

#### 2. Report Privately  
```email
To: security@kmp-learn.com

Subject: Security Vulnerability Report

**Vulnerability Type:** [e.g., SQL Injection, XSS, etc.]  
**Severity:** [Critical/High/Medium/Low]

**Description:**  
[Detailed description о the vulnerability]

**Steps к Reproduce:**
1. [Step 1]  
2. [Step 2]

**Impact:**  
[What could an attacker do?]

**Suggested Fix (optional):**  
[If you have a fix suggestion]
```

#### 3. Expected Response Time:
- **Critical:** 24 hours  
- **High:** 3 days
- **Medium:** 7 days  
- **Low:** 14 days

### If Your App is Compromised:

**Immediate Actions:**
1. **Revoke compromised credentials**  
2. **Notify affected users**  
3. **Deploy security patch**
4. **Investigate root cause**

---

## Resources

### Official Guidelines:
- [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)  
- [Android Security Guide](https://developer.android.com/training/security)
- [iOS Security Guide](https://developer.apple.com/documentation/security/)

### Libraries:
- **Android:** Jetpack Security, EncryptedSharedPreferences  
- **iOS:** Keychain, LocalAuthentication
- **KMP:** [SecureStore](https://github.com/your-repo/secure-store)

### Tools:
- **Static Analysis:** Detekt, Ktlint  
- **Dynamic Analysis:** OWASP ZAP, Burp Suite
- **Penetration Testing:** MobSF

---

## Security Updates

**We regularly update security content.** Subscribe к our security newsletter:  
[security-updates@kmp-learn.com](mailto:security-updates@kmp-learn.com)

**Security Bug Bounty:**  
We offer bounties для responsible disclosure о vulnerabilities. See [SECURITY_BOUNTY.md](./SECURITY_BOUNTY.md)

---

**Remember:** Security is a continuous process, not a one-time task! 🔒

# Security Checklist для KMP Production Deployment 🔒

## 📋 Pre-Deployment Security Audit

Используйте этот checklist перед каждым production release для обеспечения security вашего KMP приложения.

---

## 🔐 1. Data Security

### Encryption at Rest
- [ ] **Database encryption enabled**  
  - Android: SQLCipher или Jetpack Security EncryptedFile
  - iOS: Core Data encryption или SQLCipher  
  - Verification: Check database file не читается как plain text

- [ ] **Secure storage для sensitive data**
  - API tokens, refresh tokens в encrypted storage  
  - User credentials никогда не хранятся в plain text
  - PII (Personally Identifiable Information) encrypted

- [ ] **File encryption для user-generated content**
  - Images, documents uploaded by users encrypted at rest  
  - Encryption keys stored securely (не в code)

### Verification Commands:
```bash
# Android - Check if database is encrypted
sqlite3 app_database.db ".dump" | grep -i password
# Should return nothing if properly encrypted

# iOS - Check Keychain access
security dump-db | grep YOUR_BUNDLE_ID
```

---

## 🌐 2. Network Security

### TLS & Certificate Pinning
- [ ] **HTTPS only (no HTTP fallback)**
  - NetworkSecurityConfig.xml с `tlsVersion="TLSv1.2"` minimum  
  - iOS NSAppTransportSecurity с `NSAllowsArbitraryLoads = false`
  - Verification: App crashes на HTTP requests

- [ ] **Certificate pinning implemented**  
  - Pin at least 2 certificates (current + backup)
  - Pins stored в obfuscated form (не plain text в code)
  - Graceful degradation с user notification на pin failure

- [ ] **Public key pinning (preferred over certificate)**
  - More resilient to certificate renewal  
  - Use Ktor или custom interceptor для pinning

### API Security
- [ ] **All API calls use authentication**  
  - Bearer token в Authorization header
  - Token refreshed automatically перед expiration

- [ ] **Request signing для critical operations**  
  - HMAC signature для payment, data modification requests
  - Prevents request replay attacks

- [ ] **Rate limiting на client side**  
  - Prevents abuse и accidental DoS
  - Exponential backoff на rate limit errors

### Verification:
```bash
# Test with Burp Suite или mitmproxy
1. Intercept request - should see encrypted data only
2. Modify certificate - app should reject connection  
3. Replay old request - should be rejected
```

---

## 🔑 3. Authentication & Authorization

### Token Management
- [ ] **Secure token storage**  
  - Android: EncryptedSharedPreferences или Jetpack Security
  - iOS: Keychain с kSecAttrAccessibleWhenUnlockedThisDeviceOnly

- [ ] **Token expiration handling**  
  - Access token: short-lived (15-60 minutes)
  - Refresh token: longer-lived с rotation  
  - Auto-refresh перед expiration (at 80%)

- [ ] **Token invalidation на logout**  
  - Server-side token blacklist или expiration
  - Client removes all tokens от storage

### Session Security  
- [ ] **Session timeout на inactivity**
  - Configurable timeout (default: 15 minutes)  
  - User notification перед logout

- [ ] **Single session enforcement (если требуется)**
  - New login invalidates old sessions  
  - User notified если session stolen

### Biometric Authentication (если используется)
- [ ] **Biometric с fallback к password**  
  - Android: BiometricPrompt API
  - iOS: LocalAuthentication framework

- [ ] **Biometric credential storage**  
  - Android: CryptoObject с BiometricPrompt
  - iOS: LAContext с returnCode check

---

## 🛡️ 4. Application Hardening

### Code Obfuscation
- [ ] **ProGuard/R8 enabled (Android)**  
  - Keep только public API classes
  - Obfuscate class и method names
  - Verification: Decompile APK - code should be unreadable

- [ ] **Symbol stripping (iOS)**  
  - DSYMs generated для crash reports only
  - Production binary без debug symbols

### Anti-Debugging & Anti-Tampering
- [ ] **Root/Jailbreak detection**  
  - Check для common root tools (Magisk, Cydia)
  - Detect debuggers attached (ptrace на Android)
  - Graceful degradation или app exit

- [ ] **Code integrity verification**  
  - Self-check для code modification
  - Certificate validation (app signed with expected cert)

- [ ] **Native code protection**  
  - Obfuscate Kotlin/Native compiled code
  - Protect critical algorithms в native libraries

### Verification:
```bash
# Test on rooted device или with debugger attached
adb shell dumpsys activity activities | grep debuggable
# Should show false для production builds
```

---

## 📝 5. Logging & Error Handling

### Secure Logging
- [ ] **No sensitive data в logs**  
  - Never log: passwords, tokens, PII, payment info
  - Use placeholder: `user_id=***REDACTED***`

- [ ] **Build-type specific logging**  
  - Debug: verbose logging enabled
  - Production: minimal error-only logging

- [ ] **Log obfuscation (если нужно)**  
  - Custom logger что scrambles log content
  - Decryption только на server side

### Error Handling  
- [ ] **Generic error messages для users**
  - "Something went wrong" вместо "Database connection failed to server at 192.168.1.100"

- [ ] **Detailed errors только в crash reports**  
  - Sentry, Firebase Crashlytics с proper PII filtering
  - User consent для crash reporting

- [ ] **No stack traces в user-facing errors**  
  - Stack traces только в backend logs

### Verification:
```bash
# Search logs for sensitive data patterns
adb logcat | grep -i "token\|password\|secret"
# Should return nothing

# Check crash reports for PII leakage  
# Review Sentry/Firebase dashboard
```

---

## 🚫 6. Input Validation & Injection Prevention

### SQL Injection
- [ ] **Parameterized queries только**  
  - SQLDelight автоматически parameterizes
  - Never concatenate user input в SQL strings

- [ ] **Input validation для all user inputs**  
  - Whitelist allowed characters
  - Length limits enforced

### Command Injection (если есть native code)
- [ ] **No shell command execution с user input**  
  - If necessary, whitelist allowed commands only
  - Escape all parameters properly

### XSS (если есть HTML rendering)
- [ ] **Sanitize HTML content**  
  - Use library like ohtml или custom sanitizer
  - Disable dangerous tags (script, iframe)

---

## 📱 7. Platform-Specific Security

### Android
- [ ] **android:allowBackup="false"**  
  - Prevents backup с sensitive data

- [ ] **android:extractNativeLibraries="false"**  
  - Prevents native library extraction

- [ ] **android:debuggable="false" в production**  
  - Verified в decompiled APK

- [ ] **Content Provider protection**  
  - No exported ContentProviders или с proper permissions

### iOS
- [ ] **NSAllowsArbitraryLoads = false**  
  - In Info.plist

- [ ] **Keychain accessibility settings correct**  
  - kSecAttrAccessibleWhenUnlockedThisDeviceOnly для sensitive data

- [ ] **App Transport Security configured**  
  - Minimum TLS version 1.2
  - Only allowed domains specified

---

## 🔍 8. Third-Party Dependencies

### Dependency Audit
- [ ] **All dependencies up-to-date**  
  - Run `./gradlew dependencyInsight` для Android
  - Check CocoaPods или Swift Package Manager для iOS

- [ ] **No known vulnerabilities**  
  - Run Snyk, Dependabot или similar
  - Fix все High и Critical vulnerabilities

- [ ] **License compliance**  
  - Document all third-party licenses
  - Include в app если required (GPL, etc.)

### Verification:
```bash
# Check for vulnerabilities  
./gradlew assembleRelease -PskipTests
snyk test --org=your-org

# Review dependencies
./gradlew app:dependencies --configuration releaseImplementation
```

---

## 🧪 9. Security Testing

### Automated Scanning
- [ ] **Static Analysis (SAST)**  
  - Run Detekt с security ruleset
  - Ktlint для code quality

- [ ] **Dynamic Analysis (DAST)**  
  - Mobile security testing tools (MobSF, Burp Suite)
  - Automated penetration test

### Manual Testing  
- [ ] **Penetration test completed**  
  - By internal security team или external auditor
  - All Critical и High issues resolved

- [ ] **Security review by second pair of eyes**  
  - Fresh perspective catches what you miss

---

## 📊 10. Compliance & Legal

### Data Privacy
- [ ] **GDPR compliance (если EU users)**  
  - Right to access, delete data implemented
  - Data processing agreement с third parties

- [ ] **CCPA compliance (если California users)**  
  - "Do Not Sell My Data" option
  - Privacy policy updated

- [ ] **Data minimization**  
  - Collect только necessary data
  - Retention policies defined и enforced

### Permissions
- [ ] **Minimum permissions principle**  
  - Request только required permissions
  - Explain why each permission needed

- [ ] **Runtime permission handling**  
  - Proper user experience при denial
  - Graceful degradation без permission

---

## ✅ Final Pre-Deployment Checklist

### Build Configuration
- [ ] Production build type (не debug)  
- [ ] ProGuard/R8 obfuscation enabled
- [ ] Signing с production keystore/certificate  
- [ ] No debug code или log statements

### Secrets Management
- [ ] All API keys от environment variables (не hardcoded)  
- [ ] Different keys для dev/staging/production
- [ ] Keys rotated regularly (document rotation schedule)

### Monitoring & Alerting  
- [ ] Crash reporting enabled (Sentry, Firebase Crashlytics)
- [ ] Security event logging (failed auth, suspicious activity)  
- [ ] Alerts configured для error rate spikes

### Rollback Plan
- [ ] Previous version available для quick rollback  
- [ ] Database migration rollback tested (если есть)
- [ ] Feature flags для emergency disable

---

## 🚨 Security Incident Response

### If Vulnerability Discovered Post-Deployment:

1. **Assess Impact**
   - How many users affected?  
   - What data at risk?
   - Can attacker exploit remotely или needs device access?

2. **Contain**  
   - Disable vulnerable feature через feature flag (если возможно)
   - Revoke compromised tokens/keys

3. **Fix & Deploy**  
   - Hotfix prepared и tested
   - Emergency release process activated

4. **Notify**  
   - Users если their data compromised
   - App stores если required  
   - Legal/compliance team

5. **Post-Mortem**  
   - Root cause analysis
   - Process improvements чтобы prevent recurrence

---

## 📚 Additional Resources

### Tools:
- [MobSF (Mobile Security Framework)](https://mobsf.github.io/)  
- [Burp Suite](https://portswigger.net/burp)
- [Snyk](https://snyk.io/) для dependency scanning

### Guidelines:
- [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)  
- [Android Security Best Practices](https://developer.android.com/topic/security)
- [iOS App Security](https://developer.apple.com/documentation/xcodekit/securingyourapp)

### KMP-Specific:
- [Kotlin Multiplatform Security Considerations](https://www.jetbrains.com/help/kotlin-multiplatform-dev/security.html)

---

## 📝 Audit Report Template

```markdown
# Security Audit Report - [App Name] v[Version]

**Date:** [DATE]  
**Auditor:** [NAME]  
**Build:** [BUILD NUMBER]

## Summary
[Overall security posture - Good/Fair/Poor]

## Critical Issues (Must Fix Before Deployment)
1. [Issue 1] - Status: [Open/Fixed/Verified]  
2. [Issue 2] - Status: [Open/Fixed/Verified]

## High Priority Issues
1. [Issue 1] - Status: [Open/Fixed/Verified]

## Medium Priority Issues  
1. [Issue 1] - Status: [Open/Fixed/Deferred]

## Recommendations
- [Recommendation 1]
- [Recommendation 2]

## Approval
[ ] Security Team Lead: _______________ Date: _________  
[ ] Engineering Manager: _______________ Date: _________

**Deployment Approved:** [YES/NO]
```

---

**Security is a journey, not a destination. Audit regularly! 🔒🚀**

*Last Updated: [DATE]*  
*Version: 1.0*

# 📝 Самопроверка: Модуль 9 — Security в KMP

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 10+ из 14 без подглядывания.

---

## Вопрос 1. Какие API использует Android-реализация `SecureStorage` для хранения чувствительных данных?

- **A)** Обычный `SharedPreferences` в режиме `MODE_PRIVATE`
- **B)** `KeyStore "AndroidKeyStore"` вместе с `EncryptedSharedPreferences` и `MasterKey` со схемой `AES256_GCM`
- **C)** Текстовый файл во внутренней памяти приложения
- **D)** `Room` с обычным SQLite-шифрованием через SQLCipher

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В коде модуля: `KeyStore.getInstance("AndroidKeyStore")`, ключи создаются через `KeyGenParameterSpec`, а данные пишутся в `EncryptedSharedPreferences.create(...)` с `MasterKey.Builder(...).setKeyScheme(MasterKey.KeyScheme.AES256_GCM)`, схемами `AES256_SIV` для ключей и `AES256_GCM` для значений. На iOS аналог — Keychain (`kSecClassGenericPassword`).

**Почему не другие:**
- A — нешифрованные `SharedPreferences` не защищают данные
- C — файл без шифрования не является secure storage
- D — SQLCipher в модуле не рассматривается

</details>

---

## Вопрос 2. Какой transformation (`Cipher.getInstance`) используется в Android `SecureStorage`?

- **A)** `AES/ECB/PKCS5Padding`
- **B)** `RSA/ECB/PKCS1Padding`
- **C)** `AES/GCM/NoPadding`
- **D)** `DES/CBC/PKCS5Padding`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

В `save()` и `load()` android-реализации: `Cipher.getInstance("AES/GCM/NoPadding")`. Это согласовано с `KeyGenParameterSpec`: `BLOCK_MODE_GCM` и `ENCRYPTION_PADDING_NONE` (GCM — потоковый режим, в паддинге нет необходимости).

**Почему не другие:**
- A — ECB и паддинг противоречат настройкам ключа в модуле
- B — RSA здесь не применяется, ключ AES
- D — устаревший DES, в модуле не встречается

</details>

---

## Вопрос 3. Код: генерация ключа в Android Keystore

```kotlin
val keyGenParameterSpec = KeyGenParameterSpec.Builder(
    keyName,
    KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
)
    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
    .setKeySize(256)
    .build()

val keyGenerator = KeyGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore"
)
```

Какие параметры ключа задаёт этот фрагмент?

- **A)** AES 128 бит, режим CBC, паддинг PKCS7
- **B)** AES 256 бит, режим GCM, без паддинга, для шифрования и расшифрования
- **C)** RSA 2048 бит, режим GCM, для подписи
- **D)** AES 256 бит, режим ECB, паддинг отсутствует

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Модуль задаёт `PURPOSE_ENCRYPT or PURPOSE_DECRYPT`, `BLOCK_MODE_GCM`, `ENCRYPTION_PADDING_NONE`, `setKeySize(256)` и алгоритм `KEY_ALGORITHM_AES`. Это и есть связка с `AES/GCM/NoPadding` из предыдущего вопроса.

**Почему не другие:**
- A — неверные размер, режим и паддинг
- C — алгоритм AES, а не RSA; purposes тоже другие
- D — режим GCM, а не ECB

</details>

---

## Вопрос 4. Какие поля включает query для Keychain в iOS-реализации `SecureStorage`?

- **A)** `kSecClassGenericPassword`, `kSecAttrAccount` (ключ) и `kSecAttrService` (`"com.skillsync.secure"`)
- **B)** `kSecClassCertificate`, `kSecAttrLabel` и `kSecValueRef`
- **C)** Только `kSecValueData` и `kSecMatchLimitOne`
- **D)** Имя файла и путь в `Documents` директории

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Функция `getQuery(key)`:

```kotlin
mapOf(
    kSecClass to kSecClassGenericPassword,
    kSecAttrAccount to key,
    kSecAttrService to "com.skillsync.secure"
)
```

При сохранении добавляется `kSecValueData`, при загрузке — `kSecMatchLimit` и `kSecReturnData`.

**Почему не другие:**
- B — это работа с сертификатами, а не generic password
- C — это дополнительные поля, а не базовый query
- D — Keychain — не файловая система

</details>

---

## Вопрос 5. Какие статусы iOS Keychain считаются успешными при удалении элемента?

- **A)** Только `errSecSuccess`
- **B)** `errSecSuccess` и `errSecItemNotFound`
- **C)** Любой статус, отличный от `errSecSuccess`
- **D)** Только `errSecItemNotFound`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В коде:

```kotlin
status == errSecSuccess || status == errSecItemNotFound -> Result.success(Unit)
else -> Result.failure(SecurityException("Keychain delete failed: $status"))
```

Отсутствие элемента при удалении — нормальная ситуация: результат операции остаётся успешным (идемпотентность). В `save()` перед `SecItemAdd` тоже вызывается `SecItemDelete(getQuery(key))`.

**Почему не другие:**
- A — слишком строго: повторное удаление дало бы ошибку
- C — наоборот, любой неожиданный статус — провал
- D — `errSecSuccess` — как раз нормальный случай

</details>

---

## Вопрос 6. Чем отличаются атрибуты доступности в `saveWithBiometric` при `biometricRequired = true` и `false`?

- **A)** Ничем, параметр игнорируется
- **B)** `true` → `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`; `false` → `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`
- **C)** `true` → `kSecAttrAccessibleAlways`; `false` → `kSecAttrAccessibleWhenUnlocked`
- **D)** `true` → шифрование AES, `false` → шифрование RSA

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В ios-реализации:

```kotlin
this[kSecAttrAccessible] = if (biometricRequired) {
    kSecAttrAccessibleWhenUnlockedThisDeviceOnly
} else {
    kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
}
```

Оба варианта с суффиксом `ThisDeviceOnly` — данные не попадут в бэкап на другое устройство.

**Почему не другие:**
- A — параметр реально используется
- C — в модуле нет `kSecAttrAccessibleAlways`
- D — смена алгоритма шифрования здесь не происходит

</details>

---

## Вопрос 7. От какой атаки защищает certificate pinning в модуле?

- **A)** От SQL-инъекций
- **B)** От MITM (перехват трафика через подменённый сертификат)
- **C)** От reverse engineering кода
- **D)** От утечки памяти

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Критерий сдачи домашки прямо гласит: «Certificate pinning предотвращает MITM атаки». В Android-конфигурации `CertificatePinner.Builder()` привязывает домен `api.yourdomain.com` к двум SHA-256-хэшам сертификатов; если сертификат не совпадает, соединение отвергается.

**Почему не другие:**
- A — к сетевым запросам отношения не имеет (и HTTP-клиент здесь не про SQL)
- C — для этого есть обфускация и ProGuard
- D — это тема модуля производительности

</details>

---

## Вопрос 8. Код: валидация сертификата

```kotlin
private fun validateCertificate(domain: String, response: HttpResponse) {
    val pinnedHashes = pinnedCertificates[domain] ?: return
    val actualHash = extractCertificateHash(response)

    if (!pinnedHashes.contains(actualHash)) {
        throw SecurityException("Certificate pinning validation failed for $domain")
    }
}
```

Что произойдёт, если хэш сертификата не входит в список закреплённых?

- **A)** Запрос будет тихо продолжен
- **B)** Будет выброшено `SecurityException` и запрос не будет выполнен
- **C)** Хэш будет автоматически добавлен в список
- **D)** Будет выполнен повторный запрос к другому домену

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В коде явно `throw SecurityException("Certificate pinning validation failed for $domain")`. Обрати внимание и на строку `?: return`: если для домена нет закреплённых хэшей, проверка просто пропускается — то есть защита работает только для доменов, добавленных в `pinnedCertificates`.

**Почему не другие:**
- A — тихий проход означал бы отсутствие защиты
- C — список неизменяем, autoupdate хэшей опасен
- D — fallback на другой домен в модуле не описан

</details>

---

## Вопрос 9. Найди ошибку (вопрос-ловушка)

```kotlin
override suspend fun login(email: String, password: String): Result<AuthTokens> {
    val hashedPassword = hashPassword(password)
    val tokens = authApi.login(email, hashedPassword)
    // ... сохранение токенов ...
}

private fun hashPassword(password: String): String {
    // Use Argon2 or bcrypt for password hashing
    TODO("Implement secure password hashing")
}
```

Что не так с этим фрагментом?

- **A)** Всё корректно, можно отправлять в продакшен
- **B)** `hashPassword()` — заглушка `TODO`: пароль реально не хешируется, в проде нужно использовать Argon2 или bcrypt
- **C)** Нельзя хешировать пароль на клиенте — только на сервере
- **D)** Нужно вызывать `hashPassword()` после `authApi.login()`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Это вопрос-ловушка: на первый взгляд «безопасная аутентификация», но ключевая функция — `TODO("Implement secure password hashing")`. Комментарий в коде подсказывает верное решение: Argon2 или bcrypt. `TODO()` в Kotlin выбрасывает `NotImplementedError`, так что `login()` просто упадёт, а не защитит пароль.

**Почему не другие:**
- A — заглушки в security-коде нельзя оставлять в релизе
- C — модуль, наоборот, хеширует пароль перед отправкой как «дополнительную защиту»
- D — порядок в коде правильный: сначала хеш, потом запрос

</details>

---

## Вопрос 10. Как настроен фоновый обновлятор токенов в `AuthManagerImpl`?

- **A)** Проверяет каждые 30 секунд и обновляет токен за 1 минуту до истечения
- **B)** Каждые 5 минут проверяет `expires_at` и обновляет токен, если до истечения меньше 5 минут
- **C)** Обновляет токен только после получения ошибки 401
- **D)** Запускается один раз при логине и больше не работает

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`startTokenRefreshMonitor()`:

```kotlin
while (true) {
    delay(5.minutes)
    val expiresAt = secureStorage.load(EXPIRES_AT_KEY).getOrNull()?.toLongOrNull() ?: continue
    val timeUntilExpiry = expiresAt - System.currentTimeMillis()
    if (timeUntilExpiry < 5.minutes.inWholeMilliseconds) {
        refreshToken()
    }
}
```

Это соответствует критерию сдачи: «JWT tokens автоматически обновляются до expiry». При `logout()` задача отменяется через `tokenRefreshJob?.cancel()`.

**Почему не другие:**
- A — интервалы в модуле другие
- C — реактивное обновление по 401 здесь не реализовано
- D — это непрерывный цикл, а не разовый запуск

</details>

---

## Вопрос 11. Какие три ключа `AuthManagerImpl` хранит в `SecureStorage`?

- **A)** `username`, `password`, `email`
- **B)** `access_token`, `refresh_token`, `expires_at`
- **C)** `jwt`, `session_id`, `device_id`
- **D)** `api_key`, `secret`, `pin`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Константы в companion object: `ACCESS_TOKEN_KEY = "access_token"`, `REFRESH_TOKEN_KEY = "refresh_token"`, `EXPIRES_AT_KEY = "expires_at"`. В `login()` они сохраняются после успешного ответа API, в `logout()` — удаляются все трое. Пароль нигде не сохраняется.

**Почему не другие:**
- A — пароль и логин не хранятся в secure storage
- C — таких ключей в коде нет
- D — это про другую модель безопасности, здесь не используется

</details>

---

## Вопрос 12. Для чего в `proguard-rules.pro` служит строка `-keep class com.skillsync.security.** { *; }`?

- **A)** Чтобы зашифровать этот класс при сборке
- **B)** Чтобы R8/ProGuard не переименовывал и не удалял security-классы (и их члены) при обфускации
- **C)** Чтобы включить биометрию
- **D)** Чтобы добавить certificate pinning

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Директива `-keep` запрещает обфускатору трогать указанные классы. В файле рядом есть и `-dontobfuscate com.skillsync.security.SecureStorage`, `-dontobfuscate com.skillsync.auth.**`, а также `-keep` для интерфейсов, Kotlin metadata и `@Serializable`-классов — всё это нужно, чтобы reflection/сериализация и security-логика не сломались в release.

**Почему не другие:**
- A — ProGuard не занимается шифрованием
- C — биометрия настраивается через Keychain/BiometricPrompt
- D — пиннинг настраивается в сетевом клиенте

</details>

---

## Вопрос 13. Какие индикаторы рутинга/джейлбрейка проверяет `SecurityChecker`?

- **A)** Только `BuildConfig.DEBUG`
- **B)** На Android: бинарники `su`, `magisk` и `Build.FINGERPRINT` со словом `testkey`; на iOS: пути `/Applications/Cydia.app` и `/usr/bin/sshd`
- **C)** Наличие интернета и геолокации
- **D)** Размер установленного приложения и версию ОС

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

 Android-реализация через `execCommand("which su")`, `execCommand("which magisk")` и `Build.FINGERPRINT.contains("testkey")`. iOS — `File("/Applications/Cydia.app").exists() || File("/usr/bin/sshd").exists()`. Метод `isDebuggable()` отдельно возвращает `BuildConfig.DEBUG` на Android и `false` на iOS.

**Почему не другие:**
- A — это отдельный метод `isDebuggable()`, а не полная проверка
- C — сетевые/гео-условия в модуле не используются
- D — таких проверок нет

</details>

---

## Вопрос 14. Что делает `SecureApiClient.safeRequest()` перед выполнением запроса?

- **A)** Шифрует параметры запроса
- **B)** Проверяет целостность приложения и факт рутинга/джейлбрейка и возвращает `Result.failure(SecurityException(...))`, если проверка не пройдена; иначе выполняет `block()`
- **C)** Всегда обновляет access token
- **D)** Логирует все заголовки запроса с уровнем BODY

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Код:

```kotlin
if (!securityChecker.isAppIntegrityValid())
    return Result.failure(SecurityException("App integrity check failed"))
if (securityChecker.isRootedOrJailbroken())
    return Result.failure(SecurityException("Device security compromised"))
return block()
```

Это единая точка входа, через которую бизнес-запросы не выполняются на скомпрометированном устройстве или при подменённой подписи.

**Почему не другие:**
- A — шифрование здесь не выполняется
- C — обновление токена — задача `AuthManager.refreshToken()`
- D — логирование BODY есть в сетевой конфигурации, но к `safeRequest` отношения не имеет (и для security это риск утечки данных)

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 12–14 | 🎉 Отлично! Security layer для SkillSync можно собирать без подсказок |
| 9–11 | 📖 Перечитай разделы 1–3 (хранение, пиннинг, аутентификация) и просмотри вопросы, где ошибся |
| 6–8 | ⚠️ Пройди модуль ещё раз. Сфокусируйся на expect/actual для SecureStorage и на lifecycle токенов |
| < 6 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🔜 Помни: заглушки `TODO()` в security-коде — это не «потом допишем», а дыра в защите уже сейчас.

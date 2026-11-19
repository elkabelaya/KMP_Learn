# 📝 Самопроверка: Модуль 7

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 12+ из 15 без подглядывания.

---

## Вопрос 1. Сформулируй три правила паттерна Expect/Actual из начала модуля

- **A)** `expect` можно писать в любом source set; `actual` достаточно один на все платформы; сигнатуры могут отличаться
- **B)** `expect` — только в `commonMain` и только объявляет контракт; `actual` — реализация для каждой платформы; сигнатуры должны совпадать
- **C)** `expect` заменяется на `actual` во время выполнения; сигнатуры проверяются только в Kotlin/Native
- **D)** `expect` нужен только для `suspend`-функций, `actual` генерируется Gradle автоматически

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В разделе 1 прямо перечислены правила:
1. **Expect** — только в `commonMain`, объявляет контракт
2. **Actual** — реализация для каждой платформы
3. **Синхронизация:** сигнатуры должны совпадать

Пример из модуля: `expect fun getNotificationManager(): NotificationManager` в commonMain и по `actual fun getNotificationManager()` в `androidMain` / `iosMain`.

**Почему другие не подходят:**
- A — `actual` нужен для каждого target'а, а сигнатуры обязаны совпадать
- C — подстановка происходит на этапе компиляции, а не в рантайме
- D — expect может быть что угодно: функция, класс, свойство; автогенерации нет

</details>

---

## Вопрос 2. Начиная с какой версии Android для уведомлений нужно разрешение `POST_NOTIFICATIONS`?

- **A)** Начиная с Android 8 (API 26), когда появились каналы уведомлений
- **B)** Начиная с Android 13 (API 33, `Build.VERSION_CODES.TIRAMISU`); на Android 12 и ниже разрешение не требуется
- **C)** На всех версиях Android без исключений
- **D)** Только на Android 14+

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В `AndroidNotificationManager.requestPermission()`:

```kotlin
if (Build.VERSION.SDK_INT < Build.VERSION_CODES.TIRAMISU) {
    return Result.success(true)
}
```

Для Android 13+ вызывается `ActivityCompat.requestPermissions(activity, arrayOf(Manifest.permission.POST_NOTIFICATIONS), REQUEST_CODE_NOTIFICATIONS)`. `checkPermission()` тоже проверяет разрешение только для TIRAMISU+, а ниже возвращает `true`.

**Почему другие не подходят:**
- A — API 26 — это порог для **каналов** уведомлений, а не для разрешения
- C — на старых версиях разрешение не нужно
- D — порог — API 33, а не 34

</details>

---

## Вопрос 3. Как создаётся канал уведомлений в Android-реализации и зачем?

- **A)** Через `NotificationChannel(CHANNEL_ID, CHANNEL_NAME, IMPORTANCE_HIGH)` при `SDK_INT >= O` (Android 8+); канал регистрируется в блоке `init` при создании менеджера
- **B)** Канал не нужен — уведомления отправляются напрямую через `NotificationCompat.Builder`
- **C)** Через `FirebaseMessaging.getInstance().subscribeToTopic("ecotrack_channel")`
- **D)** Канал создаётся только на Android 13+ вместе с запросом разрешения

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В коде:

```kotlin
init { createNotificationChannel() }

private fun createNotificationChannel() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        val channel = NotificationChannel(
            CHANNEL_ID, CHANNEL_NAME, NotificationManager.IMPORTANCE_HIGH
        ).apply { description = "Уведомления о привычках" }
        (context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager)
            .createNotificationChannel(channel)
    }
}
```

`CHANNEL_ID = "ecotrack_channel"` затем используется в `NotificationCompat.Builder(context, CHANNEL_ID)`. Начиная с Android 8 (API 26) уведомление обязано иметь канал.

**Почему другие не подходят:**
- B — без канала уведомления на Android 8+ не покажутся
- C — это подписка на FCM-тему, а не системный канал
- D — каналы появились на API 26, разрешения — на API 33, это разные вещи

</details>

---

## Вопрос 4. Почему `requestPermission()` на Android написан через `suspendCancellableCoroutine`, а получение FCM-токена — через `.await()`?

- **A)** Потому что эти API синхронны и их нужно просто обернуть в `runBlocking`
- **B)** `ActivityCompat.requestPermissions` работает через колбэк (grantResults), а `FirebaseMessaging.token` — через Task; и то и другое нужно превратить в suspend-вызов, не блокируя поток
- **C)** Потому что `Result<Boolean>` нельзя вернуть из обычной функции
- **D)** Это требование iOS — на Android достаточно обычной функции

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Фрагмент из модуля:

```kotlin
val granted = suspendCancellableCoroutine { continuation ->
    ActivityCompat.requestPermissions(activity, arrayOf(Manifest.permission.POST_NOTIFICATIONS),
        REQUEST_CODE_NOTIFICATIONS) { _, _, grantResults ->
        val isGranted = grantResults.isNotEmpty() &&
            grantResults[0] == PackageManager.PERMISSION_GRANTED
        continuation.resume(isGranted)
    }
}
...
val token = FirebaseMessaging.getInstance().token.await()
```

`requestPermissions` сообщает результат в колбэке, а Firebase Task — в `OnCompleteListener`; `suspendCancellableCoroutine` и `.await()` (из `kotlinx-coroutines-play-services`) переводят это в мир корутин. `runBlocking` в UI-коде запрещён.

**Почему другие не подходят:**
- A — как раз наоборот: API асинхронные
- C — `Result` здесь не причина выбора механизма
- D — в Android-реализации это метод именно Android-класса

</details>

---

## Вопрос 5. Найди расхождение между контрактом и реализацией:

```kotlin
// commonMain
interface NotificationManager {
    suspend fun sendNotification(title: String, body: String): Result<Unit>
    suspend fun isPermissionGranted(): Boolean
}

// androidMain и iosMain
override suspend fun sendNotification(
    title: String,
    body: String,
    data: Map<String, String>?
): Result<Unit> { ... }
```

- **A)** Ничего, Kotlin дополнит отсутствующий параметр значением по умолчанию
- **B)** Сигнатуры не совпадают: в реализациях добавлен лишний параметр `data`, поэтому `override` не скомпилируется
- **C)** Проблема только в `isPermissionGranted` — он не должен быть `suspend`
- **D)** Проблема в типе `Result<Unit>` — он не поддерживается в commonMain

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B** (вопрос-ловушка!)

Контракт объявляет `sendNotification(title, body)`, а обе реализации (Android и iOS) добавляют третий параметр `data: Map<String, String>?`. Для `override` сигнатура должна совпадать **в точности** (значение по умолчанию здесь тоже не указано). Компилятор выдаст ошибку наподобие «'sendNotification' overrides nothing».

Отдельно стоит заметить: `suspend fun isPermissionGranted()` в интерфейсе — это осознанное решение модуля (iOS API асинхронный через completion handler; на Android синхронный `ContextCompat`, но suspend-обёртка допустима).

**Почему другие не подходят:**
- A — Kotlin не «дополняет» параметры при переопределении
- C — suspend здесь как раз корректен и пояснён комментарием в контракте
- D — `Result<Unit>` спокойно используется в commonMain

</details>

---

## Вопрос 6. Почему `getNotificationManager()` в `androidMain` бросает `NotImplementedError`, и почему `NotificationManager` впрыскивается в ViewModel через DI?

- **A)** Это баг: функцию просто забыли реализовать, DI здесь ни при чём
- **B)** Реализации нужен `Activity` (для `ActivityCompat.requestPermissions`), а у ViewModel нет доступа к Activity; менеджер должен приходить из Hilt/Koin (фабрика с Activity из Application-холдера)
- **C)** Потому что FCM-токен можно получить только в Activity
- **D)** Потому что iOS-версия тоже бросает `NotImplementedError`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В коде androidMain:

```kotlin
actual fun getNotificationManager(): NotificationManager {
    // Activity передаётся извне (Application-холдер или DI)
    throw NotImplementedError("Pass Activity via DI or Application holder")
}
```

Далее в модуле прямо объясняется: `AndroidNotificationManager` требует `Activity` для запроса разрешения, а `ViewModel` не имеет к ней доступа. Поэтому:
- **Hilt:** `@Inject` + `@AndroidEntryPoint`, Activity передаётся через модуль
- **Koin:** фабрика, получающая Activity из `Application`-холдера
- **commonMain** ничего не знает о DI и просто получает интерфейс `NotificationManager`

**Почему другие не подходят:**
- A — это осознанно спроектированная заглушка с явным сообщением
- C — получение токена (`getFcmToken()`) не требует Activity в сигнатуре
- D — iOS-версия, наоборот, возвращает `IOSNotificationManager()`

</details>

---

## Вопрос 7. Какой iOS API используется в модуле для локальных уведомлений?

- **A)** `FirebaseMessaging.messaging().subscribe(toTopic:)`
- **B)** `UNUserNotificationCenter`, `UNMutableNotificationContent`, `UNNotificationRequest`, `UNTimeIntervalTrigger` из `platform.UserNotifications` (APNs/UserNotifications)
- **C)** `UIApplication.shared.registerForRemoteNotifications()` без создания контента
- **D)** `NSFileManager.defaultManager` — тот же механизм, что и для файлов

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`IOSNotificationManager.requestPermission()` запрашивает опции `UNAuthorizationOptionAlert or Badge or Sound` через `currentNotificationCenter()`. `sendNotification()` создаёт `UNMutableNotificationContent`, `UNTimeIntervalTrigger` с интервалом `0.0`, `UNNotificationRequest` с `NSUUID().UUIDString` и добавляет запрос через `addNotificationRequest`. Все вызовы обёрнуты в `@OptIn(ExperimentalForeignApi::class)`.

**Почему другие не подходят:**
- A — FCM это Android-часть примера; для iOS здесь свой нативный API
- C — регистрация remote-уведомлений сама по себе не создаёт локальное уведомление
- D — файловая система здесь ни при чём

</details>

---

## Вопрос 8. Когда `IOSNotificationManager.isPermissionGranted()` возвращает `true`?

- **A)** Всегда — на iOS разрешение не требуется
- **B)** Когда `getNotificationSettingsWithCompletionHandler` вернул статус `UNAuthorizationStatusAuthorized` **или** `UNAuthorizationStatusProvisional`
- **C)** Только при статусе `Authorized`, `Provisional` не учитывается
- **D)** Когда `getApnsToken()` вернул `Result.success`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Код из модуля:

```kotlin
center.getNotificationSettingsWithCompletionHandler { settings, _ ->
    val granted = settings?.authorizationStatus == UNAuthorizationStatus.UNAuthorizationStatusAuthorized ||
                 settings?.authorizationStatus == UNAuthorizationStatus.UNAuthorizationStatusProvisional
    continuation.resume(granted)
}
```

Метод `suspend`, потому что iOS API асинхронный: используется `suspendCancellableCoroutine`, никакого блокирующего ожидания. `Provisional` — тихий режим уведомлений, он тоже считается разрешением.

**Почему другие не подходят:**
- A — разрешение на iOS запрашивается обязательно
- C — `Provisional` явно учтён в условии `||`
- D — `getApnsToken()` пока не реализован и возвращает `Result.failure`

</details>

---

## Вопрос 9. Как задание модуля предлагает интегрировать аналитику и краш-репорты?

- **A)** Прямо в `commonMain` через общие Kotlin-библиотеки без платформенного кода
- **B)** Через Expect/Actual: Firebase Analytics или Mixpanel для аналитики; Firebase Crashlytics или Sentry для краш-репортов
- **C)** Только на Android через Hilt, на iOS не поддерживается
- **D)** Через HTTP-клиент из Модуля 5 — events отправляются на свой сервер

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Задания 4 и 5 домашнего задания формулируют: «Интегрируйте Firebase Analytics или Mixpanel **через Expect/Actual**» и «Интегрируйте Firebase Crashlytics или Sentry **через Expect/Actual**». Отслеживать предлагается добавление привычки, завершение и синхронизацию. Это логичное продолжение паттерна из раздела 1.

**Почему другие не подходят:**
- A — у этих сервисов нативные SDK, общий контракт нужен через expect/actual
- C — обе платформы поддерживаются, просто реализация разная
- D — свой сервер здесь не описан

</details>

---

## Вопрос 10. Что входит в контракт `DeepLinkManager` и какие действия определены в `DeepLinkAction`?

- **A)** `open(url): Unit` и действия `HOME`, `SETTINGS`, `PROFILE`
- **B)** `registerDeepLinks()` и `suspend handleDeepLink(link): Result<DeepLinkAction>`; действия `OPEN_HABIT`, `ADD_HABIT`, `VIEW_STATISTICS`
- **C)** Только `handleDeepLink(link): String`, действий нет — экран выбирается строкой
- **D)** `registerDeepLinks()` и `getToken(): Result<String>`; действия `LOGIN`, `LOGOUT`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Контракт в commonMain:

```kotlin
interface DeepLinkManager {
    fun registerDeepLinks()
    suspend fun handleDeepLink(link: String): Result<DeepLinkAction>
}

enum class DeepLinkAction {
    OPEN_HABIT, ADD_HABIT, VIEW_STATISTICS
}

expect fun getDeepLinkManager(): DeepLinkManager
```

`Result<DeepLinkAction>` позволяет вернуть и неизвестный link как ошибку. Реализации → `AndroidDeepLinkManager(context)` и `IOSDeepLinkManager()`.

**Почему другие не подходят:**
- A — таких методов и действий в модуле нет
- C — строки как результат не используются, есть типизированный enum
- D — токены относятся к уведомлениям, а не к deep links

</details>

---

## Вопрос 11. Что вернёт `AndroidDeepLinkManager.handleDeepLink("https://ecotrack.app/statistics")`?

```kotlin
when (uri.path) {
    "/habit"      -> Result.success(DeepLinkAction.OPEN_HABIT)
    "/add-habit"  -> Result.success(DeepLinkAction.ADD_HABIT)
    "/statistics" -> Result.success(DeepLinkAction.VIEW_STATISTICS)
    else          -> Result.failure(Exception("Неизвестный deep link"))
}
```

- **A)** `Result.success(DeepLinkAction.OPEN_HABIT)`
- **B)** `Result.success(DeepLinkAction.VIEW_STATISTICS)`
- **C)** `Result.failure(Exception("Неизвестный deep link"))`, потому что сравнение ведётся с полным URL
- **D)** `Result.success(DeepLinkAction.ADD_HABIT)`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Код разбирает ссылку через `Uri.parse(link)` и сравнивает **`uri.path`**, то есть только путь `/statistics`, без схемы и хоста. Поэтому результат — `VIEW_STATISTICS`. Неизвестный путь попадает в `else` и даёт `Result.failure` с сообщением «Неизвестный deep link».

**Почему другие не подходят:**
- A — `/habit` даёт `OPEN_HABIT`, это другой путь
- C — сравнение идёт по `uri.path`, а не по полному URL
- D — `/add-habit` соответствует `ADD_HABIT`

</details>

---

## Вопрос 12. Чем отличается iOS-реализация deep links от Android и где она настраивается?

- **A)** iOS использует `URL(link)` и `url.path?.removePrefix("/")`, а схема объявляется в `Info.plist` через `CFBundleURLTypes` / `CFBundleURLSchemes` (`ecotrack`)
- **B)** iOS использует `Uri.parse`, а схема объявляется в `AndroidManifest.xml`
- **C)** iOS не поддерживает deep links, поэтому `IOSDeepLinkManager` — заглушка
- **D)** Настройки находятся в `file_paths.xml`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В `IOSDeepLinkManager`:

```kotlin
val url = URL(link) ?: throw Exception("Invalid URL")
when (url.path?.removePrefix("/")) {
    "habit" -> ...
    "add-habit" -> ...
    "statistics" -> ...
    else -> Result.failure(Exception("Неизвестный deep link"))
}
```

В `Info.plist` — `CFBundleURLTypes` с `CFBundleURLSchemes = ecotrack` и `CFBundleURLName = com.ecotrack.app`, плюс уже знакомый `NSAppTransportSecurity`. На Android используются `Uri.parse` и `<intent-filter>` с `android:scheme="https" android:host="ecotrack.app"` и custom scheme `ecotrack://habit`.

**Почему другие не подходят:**
- B — `Uri.parse` и манифест — это Android
- C — реализация рабочая, это не заглушка (в отличие от `getApnsToken()`)
- D — `file_paths.xml` относится к FileProvider, а не к deep links

</details>

---

## Вопрос 13. Какие методы объявляет контракт `FileManager` и что возвращает `exportData`?

- **A)** `save(data): Result<Unit>`, `load(): Result<String>`, `clear(): Result<Unit>`
- **B)** `suspend exportData(filename): Result<String>` (путь к файлу), `suspend importData(filepath): Result<Unit>`, `suspend deleteFile(filepath): Result<Unit>`; плюс `expect fun getFileManager()`
- **C)** `exportData(): ByteArray`, `importData(data: ByteArray)`, без `Result`
- **D)** Только `deleteFile(filepath): Boolean`, остальное делает система

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Контракт из commonMain:

```kotlin
interface FileManager {
    suspend fun exportData(filename: String): Result<String>  // Возвращает путь к файлу
    suspend fun importData(filepath: String): Result<Unit>
    suspend fun deleteFile(filepath: String): Result<Unit>
}

expect fun getFileManager(): FileManager
```

Все методы `suspend`, потому что файловые операции в реализациях выполняются в `withContext(Dispatchers.IO)`. `Result<String>` содержит путь/URI, которым можно поделиться.

**Почему другие не подходят:**
- A — таких имён в контракте нет
- C — возвращаемый тип `Result<String>`, а не `ByteArray`
- D — контракт описывает полный CRUD над файлом экспорта

</details>

---

## Вопрос 14. Как Android-реализация `FileManager` делится экспортированным файлом и почему?

```kotlin
val uri = FileProvider.getUriForFile(
    context, "${context.packageName}.fileprovider", file
)
Result.success(uri.toString())
```

- **A)** Возвращает обычный `file://`-путь — так проще, разрешения не нужны
- **B)** Через `FileProvider`, который даёт временный `content://` URI с разрешениями; в манифесте объявляется provider с `android:authorities="${applicationId}.fileprovider"` и `@xml/file_paths`
- **C)** Через `FirebaseMessaging` — файл отправляется как push
- **D)** Через `Uri.parse("https://ecotrack.app/export")`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Начиная с Android 7 (API 24) передача `file://` между приложениями вызывает `FileUriExposedException`. `FileProvider` выдаёт безопасный `content://` URI. Все операции обёрнуты в `withContext(Dispatchers.IO)`. В манифесте нужно объявить:

```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data android:name="android.support.FILE_PROVIDER_PATHS"
               android:resource="@xml/file_paths" />
</provider>
```

А в `res/xml/file_paths.xml` — `<files-path name="export" path="." />`.

**Почему другие не подходят:**
- A — `file://` не работает для sharing между приложениями
- C — push-уведомления не передают файлы в задании
- D — это deep link, а не путь к локальному файлу

</details>

---

## Вопрос 15. Где iOS-реализация `FileManager` ищет директорию для экспорта?

- **A)** В `context.filesDir`, как на Android
- **B)** В Documents directory: `NSFileManager.defaultManager.URLForDirectory(NSDocumentDirectory, NSDomainMask.NSUserDomainMask, ...)` через `documentsDirectory` (`by lazy`)
- **C)** В temporary directory `NSTemporaryDirectory()`
- **D)** В iCloud по умолчанию

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Код из `iosMain`:

```kotlin
private val documentsDirectory: NSURL? by lazy {
    NSFileManager.defaultManager.URLForDirectory(
        NSDocumentDirectory,
        NSDomainMask.NSUserDomainMask,
        null,
        false
    )
}

override suspend fun exportData(filename: String): Result<String> =
    withContext(Dispatchers.IO) {
        val fileUrl = documentsDirectory?.filePath?.let { "$it/$filename" }
            ?: throw Exception("Documents directory not found")
        ...
    }
```

Директория вычисляется лениво; если её найти не удалось — `Result.failure` с сообщением «Documents directory not found».

**Почему другие не подходят:**
- A — `context.filesDir` есть только на Android
- C — temporary directory в коде не используются
- D — iCloud в модуле не упоминается

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 14–15 | 🎉 Отлично! Нативные фичи и expect/actual у тебя под контролем — финишная прямая, Модуль 8 |
| 11–13 | 📖 Перечитай разделы 2–4: уведомления, deep links и файлы; посмотри вопрос 5 про сигнатуры |
| 8–10 | ⚠️ Пройди модуль ещё раз; сфокусируйся на контрактах `NotificationManager` / `DeepLinkManager` / `FileManager` и DI |
| < 8 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🔜 Квиз к Модулю 8 (финальному) — про тесты, оптимизацию и релиз в сторах.

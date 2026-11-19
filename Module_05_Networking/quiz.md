# 📝 Самопроверка: Модуль 5

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 10+ из 13 без подглядывания.

---

## Вопрос 1. Почему в модуле рекомендуется именно Ktor Client?

- **A)** OkHttp Multiplatform — потому что API уже знакомо Android-разработчикам
- **B)** Ktor Client — единый API для всех платформ, интерцепторы, нативная поддержка корутин и Kotlinx Serialization
- **C)** URLSession + OkHttp — нативные API всегда быстрее кроссплатформенных
- **D)** Retrofit — потому что это стандартный HTTP-клиент для Kotlin

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В таблице модуля Ktor Client помечен ✅ «Рекомендуется»: полная KMP-поддержка и единый API, интерцепторы (логирование, авторизация, retry), асинхронность через Coroutines и встроенная работа с Kotlinx Serialization.

**Почему другие не подходят:**
- A — OkHttp Multiplatform имеет меньше фич для iOS, подходит только для простых проектов
- C — связка нативных API даёт дублирование кода и помечена ❌ «Не рекомендуется»
- D — Retrofit в модуле даже не рассматривается

</details>

---

## Вопрос 2. В разделе «Теория» показано `HttpClient(Android)` / `HttpClient(Ios)`, а в Шаге 3 — другие движки. Какие движки реально создаются в файлах практики (`androidMain` / `iosMain`)?

- **A)** `Android` и `Ios`
- **B)** `OkHttp` и `Darwin`
- **C)** `OkHttp` и `Ios`
- **D)** `Android` и `Darwin`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B** (вопрос-ловушка!)

В разделе 2 это иллюстрация паттерна expect/actual:

```kotlin
actual fun createHttpClient() = HttpClient(Android) { ... }
actual fun createHttpClient() = HttpClient(Ios) { ... }
```

Но в **Шаге 3** («Настройка Ktor Client») код другой:

```kotlin
// androidMain
actual fun createHttpClient(): HttpClient = HttpClient(OkHttp) { ... }

// iosMain
actual fun createHttpClient(): HttpClient = HttpClient(Darwin) { ... }
```

И зависимости подтверждают это: `ktor-client-okhttp` для Android и `ktor-client-ios` (engine `darwin`) для трёх iOS-таргетов.

**Почему другие не подходят:**
- A — это из теоретического фрагмента, а не из практических файлов
- C, D — смешивают движки из разных разделов; в рабочем коде такой пары нет

</details>

---

## Вопрос 3. Куда добавляется строка `androidImplementation("io.ktor:ktor-client-okhttp:2.3.7")`?

- **A)** В `commonMain.dependencies` — чтобы клиент был доступен везде
- **B)** Только в Android-зависимости (`androidMain`), в `commonMain` лежит лишь `ktor-client-core`
- **C)** В `iosMain` — это iOS-движок
- **D)** В `settings.gradle.kts` вместе с плагином serialization

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Платформенный движок подключается только в соответствующий source set:

| Зависимость | Куда |
|-------------|------|
| `ktor-client-core`, `ktor-client-content-negotiation`, `ktor-serialization-kotlinx-json` | `commonMain` |
| `ktor-client-okhttp` | `androidImplementation` |
| `ktor-client-ios` | `iosX64Implementation`, `iosArm64Implementation`, `iosSimulatorArm64Implementation` |

В `settings.gradle.kts` добавляется **плагин** `org.jetbrains.kotlin.plugin.serialization`, а не движок.

**Почему другие не подходят:**
- A — `commonMain` не видит зависимости дочерних source set'ов
- C — OkHttp это Android
- D — это блок `pluginManagement` / `plugins`, а не зависимости движка

</details>

---

## Вопрос 4. Найди объяснение, почему `category` в DTO имеет тип `Int`:

```kotlin
@Serializable
data class HabitDto(
    val id: String,
    val title: String,
    val category: Int,  // Используем int для JSON
    val isCompleted: Boolean,
    val createdAt: Long,
    val updatedAt: Long
)
```

- **A)** Перечисления вообще нельзя использовать в Kotlin с `@Serializable`
- **B)** В JSON сохраняется `ordinal` enum'а: `category.ordinal` при маппинге в DTO и `HabitCategory.entries[this.category]` обратно
- **C)** `Int` парсится быстрее, чем `String`
- **D)** Это опечатка — на самом деле поле должно быть `HabitCategory`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Комментарий в коде говорит: «enum не сериализуется напрямую». Поэтому в `HabitEntity.toDto()` пишут `category = this.category.ordinal`, а в `HabitDto.toDomainEntity()` — `category = HabitCategory.entries[this.category]`. Это и есть причина выбора `Int`.

**Почему другие не подходят:**
- A — enum можно сериализовать (в том числе с плагином kotlinx.serialization), речь именно о формате JSON в этом примере
- C — производительность здесь ни при чём, это формат представления данных
- D — код намеренно использует `Int`, обратное преобразование есть в том же файле

</details>

---

## Вопрос 5. Что делает настройка `install(Json) { ignoreUnknownKeys = true }` в `createHttpClient()`?

- **A)** Если сервер пришлёт лишние поля JSON, они будут проигнорированы, а не вызовут ошибку парсинга
- **B)** При любом неизвестном ключе запрос упадёт с ошибкой
- **C)** Автоматически преобразует `Int` в enum
- **D)** Включает логирование всех запросов

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`ignoreUnknownKeys = true` — стандартная настройка kotlinx.serialization: DTO не должен ломаться, если API добавит новое поле. Это защищает приложение при обновлении сервера. Логирование — отдельный интерцептор `Logged`, а маппинг `Int` → enum делается вручную в extension-функции.

**Почему другие не подходят:**
- B — это как раз поведение **без** этой настройки
- C — для конвертации есть `.ordinal` / `entries[...]`
- D — за логирование отвечает `addLoggingInterceptor()`

</details>

---

## Вопрос 6. Какой фрагмент соответствует методу `createHabit` из `HabitApiRepositoryImpl`?

- **A)** `client.get("$baseUrl/habits")` → `body<HabitsListResponse>()`
- **B)** `client.post("$baseUrl/habits") { setBody(habit) }` → `body<HabitResponse>()`
- **C)** `client.put("$baseUrl/habits/$id") { setBody(habit) }` → `body<HabitResponse>()`
- **D)** `client.delete("$baseUrl/habits/$id")` → `Unit`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В коде реализация.createElement выглядит так:

```kotlin
override suspend fun createHabit(habit: HabitDto): Result<HabitDto> = safeApiCall {
    val response = client.post("$baseUrl/habits") {
        setBody(habit)
    }
    response.body<HabitResponse>()
}.map { it.data!! }
```

Итого: POST на коллекцию `/habits`, тело передаётся через `setBody`, ответ — `HabitResponse`.

**Почему другие не подходят:**
- A — это `getHabits()`, список и `HabitsListResponse`
- C — это `updateHabit(id, habit)`: PUT на `/habits/$id`
- D — это `deleteHabit(id)`

</details>

---

## Вопрос 7. Что произойдёт, когда `safeApiCall` поймает `SocketException`?

```kotlin
private inline fun <T> safeApiCall(block: () -> T): Result<T> {
    return try {
        val result = block()
        Result.success(result)
    } catch (e: io.ktor.client.network.SocketException) {
        Result.failure(NetworkException("Нет интернета"))
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

- **A)** `Result.failure(NetworkException("Нет интернета"))`
- **B)** `Result.failure` с самим `SocketException` — кастомный catch не сработает
- **C)** Приложение упадёт: исключение не обработано
- **D)** `Result.success(null)`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`catch (e: SocketException)` стоит **раньше** общего `catch (e: Exception)`, поэтому сетевая ошибка превращается в понятное доменное исключение `NetworkException("Нет интернета")`. Если бы порядок был наоборот, общий catch перехватил бы всё, и message «Нет интернета» никогда не появился бы.

**Почему другие не подходят:**
- B — Specific catch имеет приоритет над общим
- C — цель `safeApiCall` как раз в том, чтобы не крашить приложение
- D — `Result.success(null)` не возвращается ни в одной ветке

</details>

---

## Вопрос 8. В чём потенциальная опасность строчки `.map { it.data!! }` в `createHabit` / `updateHabit`?

- **A)** `!!` запрещён в Kotlin
- **B)** Если сервер вернёт `data = null`, возникнет `NullPointerException`, который `safeApiCall` превратит в `Result.failure` — ошибка API будет выглядеть как сбой
- **C)** `map` не работает с `Result`
- **D)** Опасности нет: `data` всегда не-null по контракту

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В `HabitResponse` поле `data: HabitDto? = null` — то есть оно **nullable**. `!!` бросит NPE, если сервер ответил `success = true` без данных (или с ошибкой). Это типичная ошибка: безопасный `safeApiCall` не защищает от improperly обработанного nullable-поля. Правильнее было бы явно проверить `success` и `data` и вернуть осмысленную ошибку.

**Почему другие не подходят:**
- A — `!!` допустим, но опасен
- C — `Result.map` существует и используется в коде
- D — поле `data` объявлено nullable со значением по умолчанию `null`, это опровергает данное утверждение

</details>

---

## Вопрос 9. Какой порядок действий реализован в `HabitSyncRepository.addHabit` (Offline-First)?

- **A)** Сначала запрос к API, при успехе — запись в БД
- **B)** Сначала сохранение в локальную БД; при неудаче сразу возвращаем failure; затем фоновая синхронизация с API, ошибка которой **не прерывает** выполнение (`onFailure { }`)
- **C)** Только локальная БД, сеть не используется вообще
- **D)** Только API, локальная БД обновляется при следующем запуске

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Код:

```kotlin
val localResult = localRepo.addHabit(habit)
if (localResult.isFailure) return localResult.map { }

val dto = habit.toDto()
apiRepo.createHabit(dto).onFailure { /* логирование, но не прерываем */ }
return localResult.map { }
```

Пользователь видит результат мгновенно (БД), а сеть — best-effort. Это и есть_OFFLINE-FIRST: приложение работает без интернета.

**Почему другие не подходят:**
- A — перевёрнутый порядок: при отсутствии сети данные потерялись бы
- C — тогда не было бы синхронизации
- D — противоречит коду и критерию сдачи задания 3

</details>

---

## Вопрос 10. Что на самом деле делает `syncWithServer()`?

- **A)** Очищает локальную БД и полностью загружает данные с сервера — ровно как написано в комментарии
- **B)** Получает список с сервера и **добавляет** каждый DTO в локальную БД; никакой очистки в коде нет
- **C)** Удаляет данные на сервере и оставляет локальную БД как есть
- **D)** Ничего: метод всегда возвращает `Result.failure`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B** (вопрос-ловушка!)

Комментарий в коде гласит «Очищаем локальную БД и загружаем с сервера», но на самом деле выполняется только:

```kotlin
remoteHabits.getOrNull()?.forEach { dto ->
    localRepo.addHabit(dto.toDomainEntity())
}
```

Вызова `deleteAll()` / очистки нет. Из-за этого при повторной синхронизации возможны дубликаты. Всегда сверяй комментарий с кодом.

**Почему другие не подходят:**
- A — это текст комментария, но не реальное поведение кода
- C — метод только читает API
- D — при успехе возвращается `Result.success(Unit)`

</details>

---

## Вопрос 11. Что делает `addRetryInterceptor(maxRetries = 3)`?

- **A)** Повторяет только 4xx-ошибки (например, 401)
- **B)** Повторяет только ответы 5xx: `maxRetriesCount = 3`, `retryOnServerErrors = true`, `retryOnException` проверяет диапазон `500..599`
- **C)** Повторяет любой запрос бесконечно
- **D)** Логирует тело ответа

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Логика интерцептора из модуля:

```kotlin
install(io.ktor.client.plugins.Retry) {
    maxRetriesCount = maxRetries
    retryOnServerErrors = true
    retryOnException { request, exception ->
        response.status.value in 500..599
    }
}
```

4xx (ошибки клиента) повторять бессмысленно — они не исправятся сами. Логирование — задача `Logged`.

**Почему другие не подходят:**
- A — как раз исключено: retry только для 5xx
- C — лимит `maxRetriesCount` ограничивает число попыток
- D — это другой интерцептор

</details>

---

## Вопрос 12. Какие настройки нужно добавить в манифесты по итогам модуля?

- **A)** Android: `android.permission.INTERNET`; iOS: `NSAppTransportSecurity` с `NSAllowsArbitraryLoads`
- **B)** Только `android.permission.INTERNET` — iOS ничего не требует
- **C)** `POST_NOTIFICATIONS` и `CAMERA`
- **D)** Разрешение на `READ_EXTERNAL_STORAGE` и `WRITE_EXTERNAL_STORAGE`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В конце модуля прямо сказано: не забудьте разрешения на интернет в `AndroidManifest.xml` и `Info.plist`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

`POST_NOTIFICATIONS` и камера — темы Модуля 7, а не сетевого модуля.

**Почему другие не подходят:**
- B — для iOS сетевые настройки тоже нужны
- C — относится к уведомлениям/камере
- D — к сети отношения не имеет

</details>

---

## Вопрос 13. Где в Koin регистрируется `HabitSyncRepository` и почему?

- **A)** В `networkModule` — это часть сети
- **B)** В `platformModule`: он делает `includes(databaseModule)` и `includes(networkModule)`, затем `single { HabitSyncRepository(get(), get()) }`
- **C)** В `databaseModule` — потому что данные хранятся в БД
- **D)** В `commonMain` без модуля — Koin сам найдёт конструктор

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`networkModule` создаёт `HttpClient` (с logging + retry) и `HabitApiRepositoryImpl`. А `HabitSyncRepository` объединяет **оба** мира — `HabitRepositoryImpl` (БД) и `HabitApiRepository` (сеть), поэтому он объявляется в `platformModule`, куда включены оба модуля:

```kotlin
actual val platformModule() = module {
    includes(databaseModule)
    includes(networkModule)
    single { HabitSyncRepository(get(), get()) }
}
```

**Почему другие не подходят:**
- A — в `networkModule` только клиент и API-репозиторий
- C — в `databaseModule` только локальная БД
- D — без регистрации в графе Koin зависимость не разрешится

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 12–13 | 🎉 Отлично! Сеть, Offline-First и Ktor у тебя под контролем — переходи к Модулю 6 |
| 9–11 | 📖 Перечитай шаги 4–6 (DTO, `safeApiCall`, интерцепторы), посмотри вопросы, где ошибся |
| 6–8 | ⚠️ Пройди модуль ещё раз: сфокусируйся на expect/actual для HttpClient и порядке «БД → сеть» |
| < 6 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🔜 Квиз к Модулю 6 (продвинутый UI) — про Vico, анимации и дизайн-систему.

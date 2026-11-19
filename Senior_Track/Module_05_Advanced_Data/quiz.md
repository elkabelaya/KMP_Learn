# 📝 Самопроверка: Модуль 5 — Продвинутая работа с данными

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 12+ из 15 без подглядывания.

---

## Вопрос 1. Какую проблему REST API решает GraphQL в первом разделе модуля?

- **A)** Проблему N+1: вместо трёх запросов (`/user`, `/skills`, `/sessions`) клиент одним запросом получает пользователя, навыки, сессии и имя ментора
- **B)** Проблему отсутствия типизации: REST вообще не работает с типами данных
- **C)** Проблему слишком большого размера APK: GraphQL не требует генерировать код
- **D)** Проблему авторизации: GraphQL всегда авторизован по умолчанию

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле прямо перечислены ограничения REST: 3 запроса для одного экрана (N+1), сервер возвращает лишние данные, нет real-time подписок, сложная фильтрация и пагинация. GraphQL решает это одним `query GetUserWithDetails { user(id:"123") { ... skills {...} sessions(limit:10) { mentor { name } } } }`: клиент сам описывает ровно нужный набор полей и получает всё за один round-trip.

**Почему не другие:**
- B — GraphQL даёт типобезопасность **за счёт генерации из схемы**, а REST тоже может быть типизированным (например, через OpenAPI/Kotlinx.serialization); формулировка неверна
- C — наоборот: Apollo Kotlin генерирует классы, и размер APK здесь ни при чём
- D — авторизация не является преимуществом GraphQL; это отдельная задача (headers/интерцепторы)

</details>

---

## Вопрос 2. Что нужно сделать для настройки Apollo Kotlin согласно модулю?

```kotlin
plugins {
    id("com.apollographql.apollo3") version "3.8.0"
}

kotlin {
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("com.apollographql.apollo3:apollo-runtime:3.8.0")
            }
        }
    }
}

apollo {
    service("skillSync") {
        packageName.set("com.skillsync.graphql")
        schemaFile.set(file("src/commonMain/graphql/schema.graphql"))
    }
}
```

- **A)** Подключить плагин `com.apollographql.apollo3` версии 3.8.0, добавить `apollo-runtime` в `commonMain` и сконфигурировать `service` с `schemaFile` и `packageName`
- **B)** Добавить `apollo-runtime` только в `androidMain`, потому что GraphQL — Android-библиотека
- **C)** Достаточно только положить `schema.graphql` в `commonMain/graphql` — плагин не нужен
- **D)** Нужно заменить Ktor Client на Retrofit, Apollo работает только поверх Retrofit

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Так выглядит конфигурация из content.md: плагин версии `3.8.0`, зависимость `apollo-runtime:3.8.0` именно в `commonMain` (значит код будет на всех целевых платформах), и блок `apollo { service("skillSync") { ... } }`, где задаются пакет сгенерированных классов и путь к схеме. Из схемы плагин генерирует типобезопасные `GetUserQuery`, `CreateSkillMutation`, `MessageReceivedSubscription` и т.д.

**Почему не другие:**
- B — Apollo Kotlin — multiplatform-библиотека; в `androidMain` была бы недоступна из common-кода
- C — без плагина генерация классов не произойдёт, `import com.skillsync.graphql.*` не скомпилируется
- D — Apollo — самостоятельный клиент, который приходит со своим сетевым слоем; Retrofit здесь не участвует

</details>

---

## Вопрос 3. Что делает вызов `.fetchPolicy(FetchPolicy.CacheAndNetwork)` в методе `getUserFromCache`?

- **A)** Сначала возвращает данные из нормализованного кэша Apollo, а затем в фоне обновляет их из сети
- **B)** Отправляет запрос только в сеть, полностью игнорируя кэш
- **C)** Берёт данные только из кэша и никогда не ходит в сеть
- **D)** Сохраняет результат в SQLDelight, минуя кэш Apollo

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`CacheAndNetwork` — одна из политик Apollo: первым делом отдаётся закэшированный ответ (мгновенный UI), после чего выполняется сетевой запрос, который обновляет кэш. Это компромисс между отзывчивостью и свежестью: пользователь не смотрит на спиннер, но данные в итоге актуальны. Отличается от `NetworkOnly` (только сеть) и `CacheOnly` (только кэш).

**Почему не другие:**
- B — это `FetchPolicy.NetworkOnly`
- C — это `FetchPolicy.CacheOnly`
- D — SQLDelight — это отдельный слой дискового кэша из раздела про multi-layer caching; Apollo-политика к нему отношения не имеет

</details>

---

## Вопрос 4. В схеме `schema.graphql` для real-time-событий используется отдельный тип. Какой?

- **A)** `type Subscription` с полями `messageReceived`, `sessionUpdated`, `userOnlineStatus`
- **B)** `type Query`, потому что подписка — это просто долгий запрос
- **C)** `type Mutation`, потому что сообщения создаются mutation'ами
- **D)** `scalar DateTime` — он отвечает за время событий

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В GraphQL три корневых типа: `Query` (чтение), `Mutation` (изменение), `Subscription` (поток событий). В схеме SkillSync объявлены `messageReceived(chatId: ID!): Message!`, `sessionUpdated(sessionId: ID!): Session!` и `userOnlineStatus(userId: ID!): UserStatus!`. Именно `Subscription` даёт долгоживущее соединение, по которому сервер присылает новые события.

**Почему не другие:**
- B — `Query` содержит `user`, `skills`, `skillCategories`; для потоковых обновлений он не подходит
- C — `Mutation` (`createSkill`, `scheduleSession`, `sendMessage`) — это однократные изменения, а не подписка
- D — `DateTime` — кастомный scalar для полей времени

</details>

---

## Вопрос 5. Фрагмент `GqlClient.subscribeToMessages`:

```kotlin
fun subscribeToMessages(chatId: String): Flow<Message> {
    return apolloClient
        .subscription(MessageReceivedSubscription(chatId))
        .toFlow()
        .map { response ->
            when (response) {
                is ApolloResponse.Data -> response.data.messageReceived!!
                is ApolloResponse.Exception -> throw response.exception
                else -> throw IllegalStateException("Unexpected response")
            }
        }
}
```

Что произойдёт, если сервер вернёт `ApolloResponse.Exception`?

- **A)** Будет выброшено содержащееся в ответе исключение (`response.exception`) — ошибка дойдёт до подписчика Flow
- **B)** Метод вернёт `null` вместо сообщения
- **C)** Подписка молча завершится без уведомления
- **D)** Будет повторно выполнен запрос с тем же `chatId`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В `when` явно обработаны два состояния ответа: `ApolloResponse.Data` → взять `messageReceived`, `ApolloResponse.Exception` → `throw response.exception`. Исключение, выброшенное внутри `map`, нормально передаётся downstream как ошибка Flow — подписчик сможет её перехватить через `catch`. Ветка `else` страхует от неожиданных вариантов через `IllegalStateException`.

**Почему не другие:**
- B — `!!` и `throw` здесь не дают `null`; тип возвращаемого значения — `Flow<Message>`
- C — наоборот, ошибка **не** молчаливая: она намеренно прокидывается дальше
- D — никакого повторного запроса в коде нет; reconnection — задача WebSocket-слоя, а не этого `map`

</details>

---

## Вопрос 6. Extension-функция обработки ошибок из модуля:

```kotlin
inline fun <T> result(block: () -> T): Result<T, ApolloException> =
    try {
        Result.Success(block())
    } catch (e: ApolloException) {
        Result.Failure(e)
    }
```

Какое исключение будет **поймано** этой функцией?

- **A)** Только `ApolloException` (и его наследники); любое другое исключение пробросится дальше
- **B)** Любое `Exception`, потому что `catch (e: ApolloException) — синоним `catch (e: Exception)`
- **C)** Только `IllegalStateException` из subscription
- **D)** Функция не ловит ничего — `try` здесь декоративный

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`catch` объявлен с конкретным типом `ApolloException`, значит перехватываются только ошибки Apollo (сеть, GraphQL, парсинг ответа). Если внутри `block` упадёт, скажем, `NullPointerException` от `!!` или `IllegalStateException` — он пролетит мимо этого `catch`. Возвращаемый тип `Result<T, ApolloException>` это отражает: тип ошибки зафиксирован в сигнатуре.

**Почему не другие:**
- B — типы `catch` не являются синонимами; `ApolloException` — подкласс `Exception`, а не наоборот
- C — `IllegalStateException` как раз не поймается (и в subscription он создаётся вне этой функции)
- D — `try/catch` здесь рабочий, именно он превращает исключение в `Result.Failure`

</details>

---

## Вопрос 7. Про multi-layer кэширование. Что верно про слои из модуля?

- **A)** Memory (`LruCache`) — наносекунды, ограниченный размер, теряется при перезапуске; Disk (`SQLDelight`) — миллисекунды, выживает после перезапуска; Network — секунды, свежие данные, нужен интернет
- **B)** Memory — самый медленный слой, Disk — самый быстрый, Network — persistent
- **C)** Все три слоя живут в памяти и сбрасываются при перезапуске
- **D)** Disk-слой не нужен, если включён `FetchPolicy.CacheAndNetwork`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Слои в модуле описаны так: Layer 1 `LruCache` — быстрый доступ (nanoseconds), ограниченный размер, теряется при перезапуске; Layer 2 `SQLDelight` — персистентное хранение, миллисекунды, выживает после перезапуска; Layer 3 сеть — свежие данные, секунды, требует интернета. Отсюда и порядок обращения в `DataRepository.getData`: сначала memory, потом disk, потом network.

**Почему не другие:**
- B — перепутаны скорости: память быстрее диска, диск быстрее сети
- C — именно Disk-слой и решает задачу выживания после перезапуска
- D — это разные механизмы: кэш Apollo и трёхслойный offline-first кэш сосуществуют; политика Apollo не отменяет диск

</details>

---

## Вопрос 8. Фрагмент `DataRepository.getData` (offline-first). Что происходит, когда данные найдены в disk cache и не истекли?

```kotlin
val fromDisk = diskCache.getData(id)
if (fromDisk != null && !isExpired(fromDisk)) {
    memoryCache.put(id, fromDisk)
    refreshFromNetwork(id)
    return@result fromDisk
}
```

- **A)** Данные копируются в memory cache, запускается фоновое обновление из сети, а пользователю сразу возвращаются данные с диска
- **B)** Пользователь ждёт ответа от сети, и только потом данные попадают в memory
- **C)** Данные с диска игнорируются, потому что приоритет у сети
- **D)** Происходит запись в SQLDelight и удаление из memory cache

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Это ядро offline-first подхода: свежий disk-слой — уже «достаточно хороший» ответ, поэтому он кладётся в память (`memoryCache.put`), инициируется неблокирующее `refreshFromNetwork(id)` (внутри которого ошибки просто игнорируются), и управление сразу возвращается с `fromDisk`. Пользователь видит данные мгновенно, а актуализация идёт в фоне.

**Почему не другие:**
- B — это как раз то, от чего offline-first избавляет: нет ожидания сети на пути к UI
- C — сеть вызывается только если и memory, и disk пусты/истекли; при наличии свежих disk-данных сеть не блокирует ответ
- D — наоборот: memory заполняется, а не очищается; запись в SQLDelight делает `refreshFromNetwork`, а не этот блок

</details>

---

## Вопрос 9. Какая стратегия conflict resolution названа в модуле «математически корректным разрешением конфликтов»?

- **A)** Vector Clocks
- **B)** Last Write Wins (LWW)
- **C)** Manual Merge
- **D)** FetchPolicy.CacheAndNetwork

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В разделе перечислены три стратегии: LWW — «последняя запись побеждает, просто, но может потерять данные»; Manual Merge — «пользователь сам решает, сложно, но безопасно»; Vector Clocks — «математически корректное разрешение конфликтов». Именно векторные часы отслеживают причинно-следственные связи между версиями и позволяют корректно определять, какая из правок «новее» в распределённой системе.

**Почему не другие:**
- B — LWW простая и рекомендуется модулем для 90% случаев («не усложняйте»), но она теряет данные при параллельных правках
- C — Manual Merge безопасна, но требует участия пользователя и не называется математически корректной
- D — это политика кэширования Apollo, к конфликтам отношения не имеет

</details>

---

## Вопрос 10. ⚠️ Вопрос-ловушка. Посмотри на `ConflictResolver.resolve`:

```kotlin
data class VersionedData<T>(
    val data: T,
    val version: Long, // Timestamp или incrementing counter
    val updatedAt: Instant
)

fun resolve(local: VersionedData<T>, remote: VersionedData<T>): VersionedData<T> {
    return if (remote.updatedAt > local.updatedAt) {
        remote
    } else {
        local
    }
}
```

Какое поле **фактически определяет победителя** в этой реализации?

- **A)** `updatedAt` — сравнение идёт по времени, а поле `version` в `resolve` вообще не используется
- **B)** `version` — как раз он является «Timestamp или incrementing counter»
- **C)** Размер `data`: побеждает объект с большим количеством полей
- **D)** Побеждает всегда локальная версия, потому что сравнение в коде перевёрнуто

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Ловушка в том, что `VersionedData` содержит **два** поля «версии» — `version: Long` и `updatedAt: Instant`, а комментарий называет `version` «Timestamp или incrementing counter». Однако в теле `resolve` используется **только** `remote.updatedAt > local.updatedAt`. Поле `version` здесь не читается вообще (оно появляется лишь в `merge`, где берётся `maxOf(local.version, remote.version)`). То есть это чистая стратегия Last Write Wins по времени.

**Почему не другие:**
- B — звучит правдоподобно из-за комментария, но код опровергает это: `version` в условии отсутствует
- C — размер данных никак не участвует; сравнение только по `Instant`
- D — условие не перевёрнуто: при `remote.updatedAt > local.updatedAt` побеждает `remote`, иначе — `local`; это ровно LWW

</details>

---

## Вопрос 11. В методе `ConflictResolver.merge` итоговая версия считается так:

```kotlin
return VersionedData(
    data = mergedData,
    version = maxOf(local.version, remote.version),
    updatedAt = maxOf(local.updatedAt, remote.updatedAt)
)
```

Что здесь верно?

- **A)** Берётся максимальная версия и максимальное время из двух объектов, а в `data` сохраняются объединённые `localChanges`
- **B)** Берётся минимальная версия, чтобы не «перепрыгнуть» клиентов
- **C)** `version` и `updatedAt` обнуляются — merge всегда создаёт новую версию с нуля
- **D)** В `data` остаются только remote-поля, локальные изменения теряются

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В `merge` сначала выбирается база: если `remote` новее, берётся `remote.data.copy(localChanges = local.data.localChanges + remote.data.localChanges)` (иначе `local.data`), то есть локальные правки не теряются, а объединяются. Затем метаданные выставляются как максимум из двух: `maxOf(local.version, remote.version)` и `maxOf(local.updatedAt, remote.updatedAt)`. В модуле даже написан свой `extension fun maxOf(a: Instant, b: Instant)`.

**Почему не другие:**
- B — `maxOf`, а не `minOf`: версия должна монотонно расти
- C — обнуления нет, наоборот сохраняется максимальная версия
- D — как раз наоборот: `localChanges` конкатенируются, локальные изменения сохраняются

</details>

---

## Вопрос 12. ⚠️ Вопрос-ловушка. Фрагмент `WebSocketManager`:

```kotlin
is WebSocketFrame.Close -> {
    // Соединение закрыто, reconnect
    reconnect()
}
...
private suspend fun reconnect() {
    // Реализация reconnection с exponential backoff
    delay(1000)
    connect()
}
```

Что **реально делает** `reconnect()` в этом коде, несмотря на комментарий?

- **A)** Ждёт ровно 1 секунду (`delay(1000)`) и вызывает `connect()` — это фиксированная задержка, а не exponential backoff
- **B)** Реализует экспоненциальную задержку 1 → 2 → 4 → 8 секунд, как в домашнем задании
- **C)** Немедленно переподключается без задержки
- **D)** Закрывает соединение навсегда и очищает `messageChannel`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Комментарий обещает «exponential backoff», но тело функции содержит лишь `delay(1000)` и `connect()`. То есть каждая попытка ждёт одну и ту же секунду — никакого удвоения интервала нет. Экспоненциальный backoff (1s, 2s, 4s, 8s...) и heartbeat описаны в **домашнем задании** как требования, которые ещё предстоит реализовать. Это классическая ловушка «комментарий врёт, читай код».

**Почему не другие:**
- B — именно это нужно сделать в ДЗ, но в коде отсутствует: нет ни счётчика попыток, ни вычисления `2^n`
- C — `delay(1000)` явно есть, задержка не нулевая
- D — `disconnect()` — отдельный метод; `reconnect()` не очищает канал и не «закрывает навсегда»

</details>

---

## Вопрос 13. Как `ChatService.getMessages` отделяет сообщения нужного чата от общего потока WebSocket?

```kotlin
return wsManager.incomingMessages
    .filter { it.startsWith("message:$chatId:") }
    .map { jsonStr ->
        val data = jsonStr.removePrefix("message:$chatId:")
        gson.fromJson(data, ChatMessage::class.java)
    }
```

- **A)** Фильтрует строки по префиксу `message:<chatId>:`, затем отрезает префикс и десериализует остаток через Gson в `ChatMessage`
- **B)** Отправляет на сервер запрос `GET /messages?chatId=...` и получает список
- **C)** Использует GraphQL `subscription MessageReceived` — WebSocket здесь не используется
- **D)** Сохраняет все сообщения в SQLDelight и читает оттуда по `chatId`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`incomingMessages` — общий `Flow<String>` из `messageChannel` WebSocket-менеджера. `filter` пропускает только строки с нужным префиксом, `removePrefix` убирает служебную часть, а `gson.fromJson` превращает JSON в `ChatMessage`. Симметричная операция `sendMessage` делает `gson.toJson` и шлёт `"send:$chatId:$json"` — вот как устроен простой протокол поверх WebSocket.

**Почему не другие:**
- B — это был бы REST-подход; в коде нет никакого HttpClient с GET
- C — GraphQL subscription есть в `GqlClient.subscribeToMessages`, но `ChatService` использует именно `WebSocketManager`, а не Apollo
- D — SQLDelight здесь не задействован; сообщения идут напрямую из канала в Flow

</details>

---

## Вопрос 14. Что вернёт `CompositeRepository.getData()`, если все источники (`GraphQL`, `REST`, `LocalDB`) недоступны?

```kotlin
suspend fun getData(): Result<T> {
    for ((source, fetcher) in sources) {
        try {
            val result = fetcher()
            if (result is Result.Success) {
                return Result.Success(result.data)
            }
        } catch (e: Exception) {
            continue
        }
    }
    return Result.Failure(DataUnavailableException())
}
```

- **A)** `Result.Failure(DataUnavailableException())` — после того как переберёт все источники и не найдёт успеха
- **B)** `Result.Success` с пустыми данными
- **C)** Первое же исключение будет проброшено наружу
- **D)** Метод упадёт в бесконечный цикл

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Цикл проходит по парам `(DataSource, fetcher)` в порядке списка. Возврат происходит **только** на первом `Result.Success`; исключения перехватываются и игнорируются (`continue`), равно как и `Result.Failure` без исключения — он просто не вызывает return. Если список исчерпан, возвращается `Result.Failure(DataUnavailableException())` с сообщением «All data sources are unavailable». Порядок в `SkillsRepository` задаёт приоритет: GraphQL → REST → LocalDB.

**Почему не другие:**
- B — пустых данных нигде не создаётся; `Success` возможен только если какой-то fetcher реально вернул данные
- C — `try/catch` как раз гасит исключения, чтобы перейти к следующему источнику
- D — `for` по конечному списку всегда завершается; бесконечности нет

</details>

---

## Вопрос 15. Что из перечисленного — **бонусное** задание модуля, а не обязательное требование/критерий сдачи?

- **A)** Optimistic updates — мгновенное обновление UI до подтверждения сервером
- **B)** GraphQL-запросы работают и возвращают данные (критерий сдачи)
- **C)** Кэширование работает корректно, проверено через Network Tab (критерий сдачи)
- **D)** Приложение полностью функционально в офлайн-режиме (критерий сдачи)

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В разделе «Бонусные задания» перечислены: optimistic updates, delta sync (синхронизация только изменений) и кастомный cache policy для разных типов данных. А пункты B, C, D — это прямо «Критерии сдачи» из ДЗ (там же ещё conflict resolution без потери данных и WebSocket reconnect после разрыва). Обязательные требования включают: 3 query / 2 mutation / 1 subscription, три слоя кэша с TTL memory 5 мин и disk 1 час, version tracking + LWW, WebSocket с reconnection и heartbeat, очереди на отправку офлайн.

**Почему не другие:**
- B — обязательный критерий: «GraphQL запросы работают и возвращают данные»
- C — обязательный критерий: «Кэширование работает корректно (проверьте через Network Tab)»
- D — обязательный критерий: «Приложение полностью функционально в офлайн режиме»

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 14–15 | 🎉 Отлично! GraphQL, кэш, конфликты и real-time у тебя под контролем |
| 11–13 | 📖 Неплохо — перечитай вопросы-ловушки (поле `version` и `reconnect`) и секции про кэш |
| 7–10 | ⚠️ Пройди модуль ещё раз: сфокусируйся на multi-layer кэше, LWW и fallback-репозитории |
| < 7 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🔜 Следующий модуль — Database & Persistence: транзакции SQLDelight, миграции, offline-first и Realm. Готовься к вопросам про `transaction {}` и conflict resolution.

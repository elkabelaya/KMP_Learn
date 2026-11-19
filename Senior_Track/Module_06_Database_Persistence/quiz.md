# 📝 Самопроверка: Модуль 6 — Database & Persistence

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 11+ из 14 без подглядывания.

---

## Вопрос 1. Зачем все операции с базой в `SkillsDatabaseImpl` обёрнуты в `withContext(Dispatchers.IO)`?

- **A)** Чтобы перенести блокирующие операции БД с главного потока на пул IO и не блокировать UI
- **B)** Чтобы включить транзакции SQLDelight — без `withContext` они не работают
- **C)** Чтобы сделать запросы асинхронными на уровне SQL — `withContext` заменяет `suspend`
- **D)** Чтобы автоматически закрыть соединение с БД после каждого запроса

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

SQLDelight выполняет запросы синхронно на том потоке, где они вызваны. `withContext(Dispatchers.IO)` переключает выполнение на пул потоков, подходящий для блокирующих операций ввода-вывода, и возвращает результат в исходный контекст. Это стандартный способ не «заморозить» главный поток во время чтения/записи. Транзакции при этом задаются отдельно — `database.transaction { }`.

**Почему не другие:**
- B — `transaction { }` и `withContext` — независимые вещи; транзакция работает и без смены контекста
- C — `withContext` не делает SQL асинхронным, он лишь переносит синхронный вызов на другой поток; `suspend` уже объявлен в интерфейсе
- D — закрытие соединения — задача драйвера/настроек, а не `withContext`

</details>

---

## Вопрос 2. Фрагмент `insertSkills`:

```kotlin
override suspend fun insertSkills(skills: List<Skill>) = withContext(Dispatchers.IO) {
    if (skills.isEmpty()) return@withContext

    database.transaction {
        skills.forEach { skill ->
            database.queries.insertSkill(
                id = skill.id,
                name = skill.name,
                level = skill.level.value,
                createdAt = skill.createdAt.time,
                updatedAt = skill.updatedAt.time
            ).executeAsOne()
        }
    }
}
```

Зачем пакетная вставка целиком обёрнута в одну `database.transaction { }`?

- **A)** Для атомарности: либо вставятся все элементы, ни один, плюс меньше накладных расходов, чем отдельная транзакция на каждую строку
- **B)** Чтобы `insertSkill` стал `suspend`-функцией
- **C)** Чтобы автоматически отфильтровать дубликаты по `id`
- **D)** `transaction` здесь нужна только для чтения данных обратно

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Транзакция гарантирует, что набор изменений применяется как одно целое: если на середине цикла произойдёт ошибка, все уже сделанные вставки будут откачены. Плюс движок БД не тратит ресурсы на открытие/фиксацию транзакции для каждой строки отдельно. Ранний `return@withContext` на пустом списке избавляет от лишней (пустой) транзакции.

**Почему не другие:**
- B — `suspend` уже объявлен в сигнатуре метода, транзакция на это не влияет
- C — дедупликация — задача `PRIMARY KEY` / `INSERT OR REPLACE`, а не транзакции
- D — транзакция здесь как раз вокруг **записи**; для чтения она используется в `syncWithServer`

</details>

---

## Вопрос 3. Что делает строка из `searchSkills`:

```kotlin
database.queries.searchSkills("%$query%").executeAsList().map { it.toSkill() }
```

- **A)** Оборачивает запрос в `%...%` для `LIKE`-поиска по подстроке: найдутся записи, содержащие `query` в любом месте
- **B)** Экранирует пользовательский ввод от SQL-инъекций
- **C)** Ограничивает выдачу первыми 10 результатами
- **D)** Преобразует результат в JSON

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В SQL-delight запросе, вероятно, написано что-то вроде `WHERE name LIKE ?`. Символ `%` в `LIKE` означает «любая последовательность символов», поэтому `%$query%` ищет совпадение подстроки в любом месте названия. `executeAsList()` возвращает список строк, а `.map { it.toSkill() }` преобразует их в доменные объекты.

**Почему не другие:**
- B — `%` не имеет отношения к экранированию; SQLDelight защищает от инъекций через ---
- C — лимит задавался бы в самом SQL (`LIMIT 10`), а не через `%`
- D — JSON здесь не появляется; результат сразу превращается в `Skill`

</details>

---

## Вопрос 4. Зачем в `SkillsDatabaseImpl` нужен extension `toSkill()`?

```kotlin
private fun AppDb_SkillsQueryResult.toSkill(): Skill = Skill(
    id = id(),
    name = name(),
    categoryId = categoryId(),
    level = SkillLevel.fromValue(level()),
    createdAt = Instant.fromEpochMilliseconds(createdAt()),
    updatedAt = Instant.fromEpochMilliseconds(updatedAt())
)
```

- **A)** Чтобы преобразовать «плоские» примитивы SQLDelight (`Int`, `Long`) обратно в доменные типы `SkillLevel` и `Instant`
- **B)** Чтобы открыть новое соединение с БД для каждой записи
- **C)** Чтобы выполнить `JOIN` с таблицей категорий
- **D)** Чтобы сериализовать объект в формат SQLDelight для вставки

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В базе `level` хранится как `Int` (`.value` при записи), а `createdAt`/`updatedAt` — как `Long` (`time` в миллисекундах). При чтении нужно обратное преобразование: `SkillLevel.fromValue(level())` и `Instant.fromEpochMilliseconds(...)`. Extension-функция инкапсулирует эту конвертацию и отделяет слой БД от доменной модели.

**Почему не другие:**
- B — конвертация данных не имеет отношения к соединениям
- C — `JOIN` — это часть SQL-запроса, а не маппинга результата
- D — для вставки есть обратная сторона (`skill.level.value`, `skill.createdAt.time`) в `insertSkill`, а `toSkill()` работает в читающем направлении

</details>

---

## Вопрос 5. ⚠️ Вопрос-ловушка. Как ведёт себя `AppDbMigrationHelper` при обновлении с версии 1 сразу на версию 3?

```kotlin
fun migrate(database: AppDb, oldVersion: Int, newVersion: Int) {
    when (oldVersion) {
        1 -> migrateFrom1To2(database)
        2 -> migrateFrom2To3(database)
    }
}
```

- **A)** Выполнится только `migrateFrom1To2` — `when` выбирает одну ветку по `oldVersion`, и `migrateFrom2To3` не вызовется
- **B)** Последовательно выполнятся обе миграции: 1→2, затем 2→3
- **C)** Не выполнится ни одна миграция, потому что `oldVersion != newVersion`
- **D)** Выполнится только `migrateFrom2To3`, потому что `newVersion == 3`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`when (oldVersion)` — это выбор **одной** подходящей ветки, а не цепочка. При `oldVersion == 1` сработает ветка `1 -> migrateFrom1To2(database)`, и на этом всё. Вторая ветка не выполнится, хотя приложение «пропустило» версию 2. Правильная реализация должна применять миграции последовательно (например, циклом от `oldVersion + 1` до `newVersion` или цепочкой вызовов). Аргумент `newVersion` в коде вообще не используется — ещё один признак неполноты. Это типичная ловушка «похоже на миграционную лестницу, но ею не является».

**Почему не другие:**
- B — именно это подразумевает «лестница», но в Kotlin `when` не имеет fallthrough и не повторяет проверку для изменённой версии
- C — ветка `1 ->` как раз подходит, так что миграция 1→2 выполнится
- D — `newVersion` не участвует в условии; ветка `2 ->` требует `oldVersion == 2`

</details>

---

## Вопрос 6. Что делает `migrateFrom1To2`:

```kotlin
database.transaction {
    database.execute("""
        ALTER TABLE skills
        ADD COLUMN updatedAt INTEGER NOT NULL DEFAULT 0
    """)

    val now = System.currentTimeMillis()
    database.execute("""
        UPDATE skills
        SET updatedAt = $now
        WHERE updatedAt = 0
    """)
}
```

- **A)** Добавляет колонку `updatedAt` со значением по умолчанию 0, затем заполняет её текущим временем у всех старых записей — всё в одной транзакции
- **B)** Удаляет таблицу `skills` и создаёт заново с новым полем
- **C)** Добавляет колонку, но оставляет у всех старых записей значение 0 навсегда
- **D)** Создаёт триггер, который обновляет `updatedAt` при каждой вставке

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`ALTER TABLE ... ADD COLUMN ... DEFAULT 0` добавляет новое поле, и SQLite подставляет 0 всем существующим строкам. Затем `UPDATE skills SET updatedAt = now WHERE updatedAt = 0` разово проставляет реальную метку времени старым записям. Оба оператора идут внутри `transaction {}`, чтобы база не осталась в промежуточном состоянии при ошибке.

**Почему не другие:**
- B — `ALTER TABLE ADD COLUMN` не пересоздаёт таблицу и не теряет данные; `DROP TABLE` здесь отсутствует
- C — второй `UPDATE` как раз исправляет нули, поэтому это не так
- D — `CREATE TRIGGER` в коде нет; обновление разовое, а не автоматическое

</details>

---

## Вопрос 7. Что создаёт `migrateFrom2To3`?

```kotlin
database.transaction {
    database.execute("""
        CREATE TABLE IF NOT EXISTS skill_history (
            id TEXT PRIMARY KEY,
            skillId TEXT NOT NULL,
            level INTEGER NOT NULL,
            changedAt INTEGER NOT NULL,
            FOREIGN KEY (skillId) REFERENCES skills(id)
        )
    """)
}
```

- **A)** Новую таблицу `skill_history` с первичным ключом `id` и внешним ключом на `skills(id)`, если её ещё нет
- **B)** Новую колонку `skill_history` в таблице `skills`
- **C)** Индекс по полю `level` в таблице `skills`
- **D)** Таблицу, которая удаляется при каждом запуске (`IF NOT EXISTS` означает пересоздание)

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`CREATE TABLE IF NOT EXISTS` создаёт таблицу только при отсутствии — это делает миграцию идемпотентной при повторном применении. У таблицы четыре поля, `id TEXT PRIMARY KEY`, и `FOREIGN KEY (skillId) REFERENCES skills(id)`, то есть история изменений уровня связана с исходным навыком. Всё внутри одной транзакции.

**Почему не другие:**
- B — `CREATE TABLE` создаёт отдельную таблицу, а не колонку (для колонки был бы `ALTER TABLE ... ADD COLUMN`)
- C — `CREATE INDEX` здесь не используется
- D — `IF NOT EXISTS` как раз **предотвращает** пересоздание/ошибку, если таблица уже есть; удаления нет

</details>

---

## Вопрос 8. Что делает `SkillsRepositoryImpl.getSkills()` в offline-first реализации?

```kotlin
override suspend fun getSkills(): Result<List<Skill>> {
    return try {
        // Always read from local database first (offline-first)
        val skills = database.getSkills()
        Result.success(skills)
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

- **A)** Всегда читает данные из локальной БД и возвращает их, не обращаясь к сети
- **B)** Сначала ходит на сервер, а при ошибке падает на локальную БД
- **C)** Возвращает данные только если есть активное соединение с сервером
- **D)** Запускает `syncWithServer()` перед каждым чтением

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Комментарий в коде прямо говорит «Always read from local database first». Это и есть offline-first: источник истины для чтения — локальная БД, поэтому данные доступны без интернета и без задержки сети. Обновление приходит отдельно через `syncWithServer()`, который вызывается явно (или из фонового `SyncManager`).

**Почему не другие:**
- B — это обратный порядок (network-first); в коде вызова remoteDataSource нет
- C — наоборот: сеть здесь вообще не проверяется
- D — `syncWithServer` — отдельный метод; `getSkills` его не запускает

</details>

---

## Вопрос 9. Что происходит при `addSkill` в offline-first репозитории?

```kotlin
database.insertSkill(skill)
syncManager.queueForSync(SyncOperation.Insert(skill))
Result.success(Unit)
```

- **A)** Навык сразу пишется в локальную БД, а операция вставки кладётся в очередь синхронизации (optimistic update) — пользователь не ждёт сервер
- **B)** Навык отправляется на сервер, и только после успеха сохраняется локально
- **C)** Навык сохраняется только в очереди синхронизации, в БД не попадает
- **D)** Вызывается `syncWithServer()` и результат ok возвращается пользователю

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Порядок важен: сначала мгновенная локальная запись (дальше `getSkills()` её уже увидит), затем `queueForSync(SyncOperation.Insert(skill))` — «отложенная отправка на сервер». Это и есть optimistic update из модуля: UI считает операцию выполненной сразу, а сеть догоняет в фоне. `Result.success(Unit)` возвращается без ожидания ответа сервера.

**Почему не другие:**
- B — это был бы пессимистичный подход и зависимость от сети; в коде всё наоборот
- C — локальная вставка есть (`database.insertSkill`), очередь — дополнение, а не замена
- D — `syncWithServer()` здесь не вызывается; его запускает `SyncManager`

</details>

---

## Вопрос 10. В `syncWithServer` есть три ветки разрешения конфликтов. Что верно?

```kotlin
when {
    localSkill == null -> {
        database.insertSkill(remoteSkill)
    }
    remoteSkill.updatedAt.isAfter(localSkill.updatedAt) -> {
        database.updateSkill(remoteSkill)
    }
    else -> {
        syncManager.queueForSync(SyncOperation.Upload(localSkill))
    }
}
```

- **A)** Нет записи → вставить remote; remote новее → обновить локальную; иначе (локальная новее) → поставить локальную в очередь на отправку серверу
- **B)** Записи нет → ничего не делать; remote новее → удалить локальную; иначе → обновить сервер из локальной
- **C)** Всегда побеждает сервер, локальные изменения игнорируются
- **D)** Всегда побеждает локальная версия, remote-данные не сохраняются

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Это LWW-стратегия по `updatedAt` с учётом трёх случаев. Если локальной копии нет — просто вставляем пришедшую с сервера. Если серверская запись свежее — перезаписываем локальную. В оставшемся случае локальная новее — значит сервер «отстаёт», и локальную версию нужно `queueForSync(SyncOperation.Upload(localSkill))`, чтобы догнать. В конце выставляется `_lastSyncTime.value = Instant.now()` для инкрементальной синхронизации (`getSkillsSince(lastSync)`).

**Почему не другие:**
- B — при `localSkill == null` вставка всё-таки происходит; удаления в коде нет
- C — локальная версия не игнорируется: в `else` её отправляют на сервер
- D — при более свежей remote-записи локальная обновляется, так что «всегда побеждает локальная» неверно

</details>

---

## Вопрос 11. Как устроен `SyncManager` для фоновой синхронизации?

```kotlin
private val syncQueue = Channel<SyncOperation>(Channel.UNLIMITED)

sealed class SyncOperation {
    data class Insert(val skill: Skill) : SyncOperation()
    data class Update(val skill: Skill) : SyncOperation()
    data class Delete(val id: String) : SyncOperation()
    data class Upload(val skill: Skill) : SyncOperation()
}

fun queueForSync(operation: SyncOperation) {
    syncQueue.trySend(operation)
}

fun startBackgroundSync() {
    coroutineScope.launch(Dispatchers.IO) {
        syncQueue.consumeEach { operation ->
            processSyncOperation(operation)
        }
    }
}
```

- **A)** Это очередь операций с неблокирующей отправкой `trySend`; отдельная корутина на `Dispatchers.IO` последовательно разбирает её через `consumeEach`
- **B)** Это `StateFlow`, который хранит только последнюю операцию и перезаписывает предыдущие
- **C)** Операции выполняются немедленно в потоке вызвавшего, без очереди
- **D)** `Channel.UNLIMITED` ограничивает очередь ровно одной операцией

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`Channel<SyncOperation>(Channel.UNLIMITED)` — неограниченная канал-очередь. `trySend` — неблокирующая отправка без приостановки корутины. `startBackgroundSync()` запускает корутину на IO, которая `consumeEach` читает операции одну за другой и вызывает `processSyncOperation` (в коде стоит комментарий про retry и exponential backoff — это место для доработки). `sealed class` даёт исчерпывающую обработку четырёх типов операций.

**Почему не другие:**
- B — `Channel` накапливает **все** операции, а не только последнюю; `StateFlow` здесь не используется
- C — вызов `queueForSync` лишь кладёт операцию в канал; выполнение идёт в отдельной корутине
- D — `UNLIMITED` означает отсутствие ограничения размера; это противоположность «ровно одна»

</details>

---

## Вопрос 12. Для чего в `SkillsRepository` объявлены `isSyncing: StateFlow<Boolean>` и `lastSyncTime: StateFlow<Instant?>`?

- **A)** Чтобы UI мог наблюдать за статусом синхронизации: `isSyncing` становится `true` в начале `syncWithServer` и `false` в `finally`, а `lastSyncTime` обновляется после успешной синхронизации и используется для инкрементальной загрузки
- **B)** Чтобы хранить настройки подключения к серверу
- **C)** Чтобы заменять собой локальную базу данных
- **D)** Чтобы считать количество уже синхронизированных элементов

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В реализации `_isSyncing = MutableStateFlow(false)` выставляется в `true` в начале `syncWithServer()` и сбрасывается в `false` в блоке `finally` — значит статус корректен даже при ошибке. `_lastSyncTime` изначально `null`, после успеха получает `Instant.now()` и передаётся в `remoteDataSource.getSkillsSince(lastSync)`, реализуя инкрементальную синхронизацию. Оба потока наружу отдаются как `asStateFlow()` (read-only).

**Почему не другие:**
- B — это не конфигурация, а наблюдаемое состояние процесса
- C — база данных остаётся `SkillsDatabase`; StateFlow её не заменяет
- D — счётчик элементов в коде отсутствует; `Instant?` — это время, а не количество

</details>

---

## Вопрос 13. Как конфигурация Realm сделана кроссплатформенной?

```kotlin
// commonMain
expect fun createRealmConfiguration(): RealmConfiguration

// androidMain / iosMain
actual fun createRealmConfiguration(): RealmConfiguration {
    return RealmConfiguration.Builder(SchemaModule)
        .name("skillsync.realm")
        .build()
}
```

- **A)** В `commonMain` объявлен `expect`-контракт, а в `androidMain` и `iosMain` — `actual`-реализации с общим `SchemaModule` и именем файла `skillsync.realm`
- **B)** Одна реализация в `commonMain` используется на обеих платформах без `actual`
- **C)** На iOS Realm вообще не используется — только на Android
- **D)** Конфигурация создаётся динамически из GraphQL-схемы

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`RealmConfiguration` — платформо-зависимый объект, поэтому в common-коде объявляется только `expect fun createRealmConfiguration(): RealmConfiguration`, а каждая платформа предоставляет свою `actual`-реализацию (в модуле они почти идентичны: `RealmConfiguration.Builder(SchemaModule).name("skillsync.realm").build()`). Это тот же механизм expect/actual, что и в модуле по UI, но применённый к БД.

**Почему не другие:**
- B — без `actual` для каждого target'а common-код не скомпилируется
- C — в content.md есть и `androidMain`, и `iosMain` реализации, значит Realm используется на обеих платформах
- D — GraphQL-схема из предыдущего модуля здесь не участвует; конфигурация строится из `SchemaModule`

</details>

---

## Вопрос 14. Фрагмент репозитория на Realm:

```kotlin
override suspend fun addSkill(skill: Skill): Result<Unit> {
    return try {
        realm.writeBlocking {
            copyToRealm(skill.toEntity())
        }
        Result.success(Unit)
    } catch (e: Exception) {
        Result.failure(e)
    }
}

fun observeSkills(): Flow<List<Skill>> {
    return realm.where<SkillEntity>()
        .findAll()
        .asFlow()
        .map { results -> results.map { it.toSkill() } }
}
```

Что верно про этот код?

- **A)** Запись идёт через `writeBlocking { copyToRealm(...) }`, а `observeSkills()` возвращает живой Flow от Realm (`asFlow()`), который эмитит обновлённый список при изменениях в базе
- **B)** `asFlow()` — это обычный `flowOf` снимка: новые изменения в Realm не приходят
- **C)** `writeBlocking` асинхронно отправляет данные на сервер
- **D)** `copyToRealm` копирует объект в SQLDelight, а не в Realm

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`realm.writeBlocking { }` открывает транзакцию записи и `copyToRealm(skill.toEntity())` сохраняет сущность (в `SkillEntity` есть поля `isSynced` и `lastSyncAt`, специфичные для Realm). `realm.where<SkillEntity>().findAll().asFlow()` — это live query: Realm сам присылает обновлённые результаты при любых изменениях, и `.map { ... toSkill() }` преобразует их в доменные объекты. Плюс `getSkills()` использует `.sorted("createdAt")`.

**Почему не другие:**
- B — `asFlow()` у Realm Results как раз и есть реактивный «живой» запрос, а не статичный снимок
- C — `writeBlocking` работает с локальной базой Realm; отправка на сервер — задача `SyncManager` из предыдущего раздела
- D — `copyToRealm` — метод Realm; SQLDelight в этом классе не используется

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 13–14 | 🎉 Отлично! Транзакции, миграции, offline-first и Realm — твоя уверенная зона |
| 10–12 | 📖 Хорошо — обнови вопросы-ловушки про `when` в миграциях и порядок в `syncWithServer` |
| 6–9 | ⚠️ Пройди модуль ещё раз: сфокусируйся на миграциях и трёх ветках разрешения конфликтов |
| < 6 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🔜 Дальше — advanced networking с Ktor, WebSocket и GraphQL. Пригодятся навыки из модуля по данным и этого модуля.

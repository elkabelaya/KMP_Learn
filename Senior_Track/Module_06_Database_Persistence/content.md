# 📘 Модуль 3: Database & Persistence

**В этом модуле вы освоите advanced database patterns для KMP: SQLDelight transactions, Realm, offline-first архитектура и data synchronization.**

**Цели модуля:**
1. Освоить advanced SQLDelight features (transactions, migrations)
2. Интегрировать Realm для NoSQL use cases
3. Реализовать offline-first архитектуру с background sync
4. Настроить data synchronization между устройствами

**Время выполнения:** ~30 часов.

---

## 1. SQLDelight Advanced Features

### Transactions и Migrations:

```kotlin
// commonMain - Database interface
interface SkillsDatabase {
    suspend fun insertSkill(skill: Skill)
    suspend fun getSkills(): List<Skill>
    suspend fun updateSkill(skill: Skill)
    suspend fun deleteSkill(id: String)
    
    // Batch operations
    suspend fun insertSkills(skills: List<Skill>)
    suspend fun deleteSkills(ids: List<String>)
    
    // Complex queries
    suspend fun searchSkills(query: String): List<Skill>
    suspend fun getSkillsByCategory(categoryId: String): List<Skill>
}

// androidMain - SQLDelight implementation
class SkillsDatabaseImpl(
    private val database: AppDb
) : SkillsDatabase {
    
    override suspend fun insertSkill(skill: Skill) = withContext(Dispatchers.IO) {
        database.transaction {
            database.queries.insertSkill(
                id = skill.id,
                name = skill.name,
                categoryId = skill.categoryId,
                level = skill.level.value,
                createdAt = skill.createdAt.time,
                updatedAt = skill.updatedAt.time
            ).executeAsOne()
        }
    }
    
    override suspend fun getSkills(): List<Skill> = withContext(Dispatchers.IO) {
        database.queries.getSkills().executeAsList().map { it.toSkill() }
    }
    
    override suspend fun insertSkills(skills: List<Skill>) = withContext(Dispatchers.IO) {
        if (skills.isEmpty()) return@withContext
        
        database.transaction {
            skills.forEach { skill ->
                database.queries.insertSkill(
                    id = skill.id,
                    name = skill.name,
                    categoryId = skill.categoryId,
                    level = skill.level.value,
                    createdAt = skill.createdAt.time,
                    updatedAt = skill.updatedAt.time
                ).executeAsOne()
            }
        }
    }
    
    override suspend fun searchSkills(query: String): List<Skill> = withContext(Dispatchers.IO) {
        database.queries.searchSkills("%$query%").executeAsList().map { it.toSkill() }
    }
    
    // Extension function для конвертации
    private fun AppDb_SkillsQueryResult.toSkill(): Skill = Skill(
        id = id(),
        name = name(),
        categoryId = categoryId(),
        level = SkillLevel.fromValue(level()),
        createdAt = Instant.fromEpochMilliseconds(createdAt()),
        updatedAt = Instant.fromEpochMilliseconds(updatedAt())
    )
}

// Migrations
class AppDbMigrationHelper {
    fun migrate(database: AppDb, oldVersion: Int, newVersion: Int) {
        when (oldVersion) {
            1 -> migrateFrom1To2(database)
            2 -> migrateFrom2To3(database)
        }
    }
    
    private fun migrateFrom1To2(database: AppDb) {
        database.transaction {
            // Add new column
            database.execute("""
                ALTER TABLE skills 
                ADD COLUMN updatedAt INTEGER NOT NULL DEFAULT 0
            """)
            
            // Update existing records
            val now = System.currentTimeMillis()
            database.execute("""
                UPDATE skills 
                SET updatedAt = $now 
                WHERE updatedAt = 0
            """)
        }
    }
    
    private fun migrateFrom2To3(database: AppDb) {
        database.transaction {
            // Create new table for skill history
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
    }
}
```

### Offline-First Architecture:

```kotlin
// commonMain - Repository с offline-first
interface SkillsRepository {
    suspend fun getSkills(): Result<List<Skill>>
    suspend fun addSkill(skill: Skill): Result<Unit>
    suspend fun syncWithServer(): Result<List<Skill>>
    
    // Sync status
    val isSyncing: StateFlow<Boolean>
    val lastSyncTime: StateFlow<Instant?>
}

class SkillsRepositoryImpl(
    private val database: SkillsDatabase,
    private val remoteDataSource: RemoteSkillsDataSource,
    private val syncManager: SyncManager
) : SkillsRepository {
    
    private val _isSyncing = MutableStateFlow(false)
    override val isSyncing: StateFlow<Boolean> = _isSyncing.asStateFlow()
    
    private val _lastSyncTime = MutableStateFlow<Instant?>(null)
    override val lastSyncTime: StateFlow<Instant?> = _lastSyncTime.asStateFlow()
    
    override suspend fun getSkills(): Result<List<Skill>> {
        return try {
            // Always read from local database first (offline-first)
            val skills = database.getSkills()
            Result.success(skills)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override suspend fun addSkill(skill: Skill): Result<Unit> {
        return try {
            // Write to local database immediately
            database.insertSkill(skill)
            
            // Mark for sync (optimistic update)
            syncManager.queueForSync(SyncOperation.Insert(skill))
            
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override suspend fun syncWithServer(): Result<List<Skill>> {
        _isSyncing.value = true
        
        return try {
            // Get last sync time for incremental sync
            val lastSync = _lastSyncTime.value
            
            // Fetch updates from server
            val remoteSkills = remoteDataSource.getSkillsSince(lastSync)
            
            // Merge with local database (conflict resolution)
            database.transaction {
                remoteSkills.forEach { remoteSkill ->
                    val localSkill = database.getSkillById(remoteSkill.id)
                    
                    when {
                        localSkill == null -> {
                            // New skill - insert
                            database.insertSkill(remoteSkill)
                        }
                        remoteSkill.updatedAt.isAfter(localSkill.updatedAt) -> {
                            // Remote is newer - update local
                            database.updateSkill(remoteSkill)
                        }
                        else -> {
                            // Local is newer - upload to server
                            syncManager.queueForSync(SyncOperation.Upload(localSkill))
                        }
                    }
                }
            }
            
            _lastSyncTime.value = Instant.now()
            Result.success(getSkills().getOrNull())
        } catch (e: Exception) {
            Result.failure(e)
        } finally {
            _isSyncing.value = false
        }
    }
}

// Sync Manager для background sync
class SyncManager(
    private val coroutineScope: CoroutineScope
) {
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
    
    private suspend fun processSyncOperation(operation: SyncOperation) {
        // Implement sync logic with retry, exponential backoff, etc.
    }
}
```

---

## 2. Realm для NoSQL Use Cases

```kotlin
// commonMain - Realm configuration
expect fun createRealmConfiguration(): RealmConfiguration

// androidMain
actual fun createRealmConfiguration(): RealmConfiguration {
    return RealmConfiguration.Builder(SchemaModule)
        .name("skillsync.realm")
        .build()
}

// iosMain  
actual fun createRealmConfiguration(): RealmConfiguration {
    return RealmConfiguration.Builder(SchemaModule)
        .name("skillsync.realm")
        .build()
}

// Realm entity
@RealmModel
class SkillEntity {
    @PrimaryKey var id: String = ""
    var name: String = ""
    var categoryId: String = ""
    var level: Int = 0
    var createdAt: Long = 0
    var updatedAt: Long = 0
    
    // Realm-specific fields
    var isSynced: Boolean = false
    var lastSyncAt: Long? = null
}

// Repository с Realm
class SkillsRepositoryWithRealm(
    private val realm: Realm
) : SkillsRepository {
    
    override suspend fun getSkills(): Result<List<Skill>> {
        return try {
            val skills = realm.where<SkillEntity>()
                .findAll()
                .sorted("createdAt")
                .map { it.toSkill() }
            
            Result.success(skills)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
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
    
    // Realm-specific: Live queries с реактивностью
    fun observeSkills(): Flow<List<Skill>> {
        return realm.where<SkillEntity>()
            .findAll()
            .asFlow() // Realm live query
            .map { results -> 
                results.map { it.toSkill() }
            }
    }
}

// Extension functions
fun SkillEntity.toSkill(): Skill = Skill(
    id = id,
    name = name,
    categoryId = categoryId,
    level = SkillLevel.fromValue(level),
    createdAt = Instant.fromEpochMilliseconds(createdAt),
    updatedAt = Instant.fromEpochMilliseconds(updatedAt)
)

fun Skill.toEntity(): SkillEntity = SkillEntity().apply {
    this.id = id
    this.name = name
    this.categoryId = categoryId
    this.level = level.value
    this.createdAt = createdAt.epochSeconds * 1000
    this.updatedAt = updatedAt.epochSeconds * 1000
}
```

---

## 📝 Домашнее задание (Модуль 3)

### Задача: Реализация offline-first data layer для SkillSync

**Требования:**
1. Настройте SQLDelight с migrations (3+ версии)
2. Реализуйте offline-first repository с background sync
3. Добавьте conflict resolution strategy (last-write-wins)
4. Напишите unit-тесты для database operations

**Критерии сдачи:**
- ✅ Database работает на Android и iOS
- ✅ Migrations успешно применяются
- ✅ Offline-first: данные доступны без интернета
- ✅ Background sync с retry logic

---

**Следующий модуль:** В Module_04 мы изучим advanced networking с Ktor, WebSocket и GraphQL.

Удачи! 🚀

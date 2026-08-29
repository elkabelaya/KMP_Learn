# 📘 Модуль 1: Advanced KMP Fundamentals

**Добро пожаловать в первый модуль Senior Track!**  
В этом модуле вы углубите понимание KMP internals, изучите продвинутые возможности expect/actual, custom build configurations и performance optimization.

**Цели модуля:**
1. Понять internals KMP compilation process
2. Освоить advanced expect/actual patterns
3. Настроить custom build configurations
4. Оптимизировать performance KMP приложений

**Время выполнения:** ~20 часов (3-4 недели).

---

## 1. KMP Internals: Как это работает

### Процесс компиляции KMP:

```
Source Code (Kotlin)
       ↓
    [Compilation]
       ↓
┌───────────────┬───────────────┬───────────────┐
│   JVM Target  │  JS/JSR Target│ Native Target │
│ (JAR files)   │ (JavaScript)  │ (Binary)      │
└───────────────┴───────────────┴───────────────┘
       ↓                ↓                ↓
  Android App      Web App        iOS App
```

### Source Set Hierarchy:

```
commonMain (базовый код для всех платформ)
       ↓
┌──────┴───────────────┬──────────────────────┐
│                      │                      │
jvmMain              jsMain               nativeMain
(Android, JVM)       (Web)           (iOS, macOS, etc.)
       ↓                      ↓                      ↓
┌──────┴───────┐          ┌──┴──┐        ┌─────────┴─────────┐
│              │          │     │        │                   │
androidMain  jvmDesktop  jsIr jsJs   iosX64            iosArm64
```

### Expect/Actual механизм:

**Как работает:**
1. В `commonMain` объявляете `expect` функцию/класс
2. В platform-specific source sets реализуете `actual`
3. Компилятор подставляет правильную реализацию для каждой платформы

**Продвинутые паттерны:**

#### Pattern 1: Expect/Actual для классов

```kotlin
// commonMain
expect open class PlatformStorage {
    suspend fun save(key: String, value: String): Boolean
    suspend fun load(key: String): String?
    suspend fun delete(key: String): Boolean
}

// androidMain
actual class PlatformStorage actual constructor() {
    private val prefs = androidContext().getSharedPreferences("storage", MODE_PRIVATE)
    
    actual suspend fun save(key: String, value: String): Boolean = withContext(Dispatchers.IO) {
        prefs.edit().putString(key, value).apply()
        true
    }
    
    actual suspend fun load(key: String): String? = withContext(Dispatchers.IO) {
        prefs.getString(key, null)
    }
    
    actual suspend fun delete(key: String): Boolean = withContext(Dispatchers.IO) {
        prefs.edit().remove(key).apply()
        true
    }
}

// iosMain
actual class PlatformStorage actual constructor() {
    private val defaults = NSUserDefaults.standardUserDefaults
    
    actual suspend fun save(key: String, value: String): Boolean = withContext(Dispatchers.Default) {
        defaults.setObject(value, key)
        true
    }
    
    actual suspend fun load(key: String): String? = withContext(Dispatchers.Default) {
        defaults.stringForKey(key) as? String
    }
    
    actual suspend fun delete(key: String): Boolean = withContext(Dispatchers.Default) {
        defaults.removeObject(key)
        true
    }
}
```

#### Pattern 2: Expect/Actual для sealed classes (Platform-specific enums)

```kotlin
// commonMain - объявляем expect sealed class
expect sealed class PlatformBiometricType {
    companion object {
        fun getAvailableTypes(): List<PlatformBiometricType>
    }
}

// androidMain
actual sealed class PlatformBiometricType {
    actual companion object {
        actual fun getAvailableTypes(): List<PlatformBiometricType> = listOf(
            Fingerprint, 
            FaceRecognition
        )
    }
    
    actual object Fingerprint : PlatformBiometricType()
    actual object FaceRecognition : PlatformBiometricType()
}

// iosMain  
actual sealed class PlatformBiometricType {
    actual companion object {
        actual fun getAvailableTypes(): List<PlatformBiometricType> = listOf(
            FaceID,
            TouchID
        )
    }
    
    actual object FaceID : PlatformBiometricType()
    actual object TouchID : PlatformBiometricType()
}
```

#### Pattern 3: Expect/Actual для extension функций

```kotlin
// commonMain
expect fun String.platformEncode(): String
expect fun String.platformDecode(): String

// androidMain
actual fun String.platformEncode(): String = 
    android.util.Base64.encodeToString(toByteArray(), android.util.Base64.NO_WRAP)

actual fun String.platformDecode(): String = 
    String(android.util.Base64.decode(toByteArray(), android.util.Base64.NO_WRAP))

// iosMain
actual fun String.platformEncode(): String = 
    toByteArray().platform.toBase64String()

actual fun String.platformDecode(): String = 
    String(kotlinx.cinterop.byteArrayOf(*toByteArray().platform.fromBase64String()))
```

---

## 2. Custom Build Configurations

### Multi-target configuration:

Создайте `shared/build.gradle.kts`:

```kotlin
plugins {
    kotlin("multiplatform")
    id("com.android.library")
}

kotlin {
    // Android target
    androidTarget {
        publishLibraryVariants("release", "debug")
        
        compilations.all {
            kotlinOptions {
                jvmTarget = "17"
            }
        }
    }
    
    // iOS targets
    iosX64()
    iosArm64()
    iosSimulatorArm64()
    
    // macOS targets (для desktop app)
    macosX64()
    macosArm64()
    
    // Linux target (для backend)
    linuxX64()
    
    // Windows target
    mingwX64()
    
    // Apply custom hierarchy
    applyDefaultHierarchyTemplate {
        common {
            group("mobile") {
                withAndroidTarget()
                withIosTargets()
            }
            
            group("desktop") {
                withMacosTargets()
                linuxX64()
                mingwX64()
            }
        }
    }
    
    sourceSets {
        val commonMain by getting
        
        // Mobile-specific code (Android + iOS)
        val mobileMain by creating {
            dependsOn(commonMain)
        }
        
        val androidMain by getting {
            dependsOn(mobileMain)
        }
        
        val iosX64Main by creating {
            dependsOn(mobileMain)
        }
        
        val iosArm64Main by creating {
            dependsOn(mobileMain)
        }
        
        val iosSimulatorArm64Main by creating {
            dependsOn(mobileMain)
        }
        
        // Desktop-specific code (macOS + Linux + Windows)
        val desktopMain by creating {
            dependsOn(commonMain)
        }
        
        val macosX64Main by creating {
            dependsOn(desktopMain)
        }
        
        val macosArm64Main by creating {
            dependsOn(desktopMain)
        }
        
        val linuxX64Main by creating {
            dependsOn(desktopMain)
        }
        
        val mingwX64Main by creating {
            dependsOn(desktopMain)
        }
    }
}

// Custom task для cross-compilation
tasks.register("buildAllPlatforms") {
    group = "build"
    description = "Builds for all targets"
    
    dependsOn(
        tasks.named("assembleRelease"),
        tasks.named("iosX64Binaries"),
        tasks.named("iosArm64Binaries"),
        tasks.named("macosX64Binaries"),
        tasks.named("linuxX64Binaries")
    )
}
```

### Build flavors для KMP:

Создайте `shared/build.gradle.kts` (product flavors):

```kotlin
android {
    flavorDimensions += "environment"
    
    productFlavors {
        create("development") {
            dimension = "environment"
            applicationId = "com.skillsync.dev"
        }
        
        create("staging") {
            dimension = "environment"
            applicationId = "com.skillsync.staging"
        }
        
        create("production") {
            dimension = "environment"
            applicationId = "com.skillsync"
        }
    }
}

kotlin {
    sourceSets {
        // Environment-specific source sets
        val commonDevelopment by creating {
            dependsOn(commonMain)
        }
        
        val commonStaging by creating {
            dependsOn(commonMain)
        }
        
        val commonProduction by creating {
            dependsOn(commonMain)
        }
    }
}

// Map Android flavors to KMP source sets
android {
    flavorDimensions += "environment"
    
    productFlavors.all { flavor ->
        kotlin.sourceSets.create("${flavor.name}Common") {
            dependsOn(kotlin.sourceSets.getByName("commonMain"))
        }
    }
}
```

---

## 3. Performance Optimization

### Benchmarking KMP код:

Создайте `shared/benchmark/src/commonMain/kotlin/Benchmarks.kt`:

```kotlin
package benchmarks

import kotlinx.coroutines.*
import kotlin.time.*

class PerformanceBenchmark {
    
    fun benchmarkBlock(name: String, iterations: Int = 1000, block: () -> Unit): BenchmarkResult {
        println("Benchmarking: $name ($iterations iterations)")
        
        // Warm-up runs
        repeat(100) { block() }
        
        // Actual benchmark
        val startTime = TimeSource.Monotonic.markNow()
        
        repeat(iterations) {
            block()
        }
        
        val endTime = TimeSource.Monotonic.markNow()
        val duration = startTime.elapsedNow()
        
        val result = BenchmarkResult(
            name = name,
            iterations = iterations,
            totalDuration = duration,
            averagePerIteration = duration / iterations,
            throughput = iterations / duration.inSeconds
        )
        
        println(result)
        return result
    }
    
    data class BenchmarkResult(
        val name: String,
        val iterations: Int,
        val totalDuration: Duration,
        val averagePerIteration: Duration,
        val throughput: Double // operations per second
    ) {
        override fun toString(): String = 
            "$name: ${totalDuration.inMilliseconds}ms total, " +
            "${averagePerIteration.inNanoseconds}ns avg, " +
            "%.2f ops/sec".format(throughput)
    }
}

// Example usage:
fun main() = runBlocking {
    val benchmark = PerformanceBenchmark()
    
    // Benchmark string operations
    benchmark.benchmarkBlock("String concatenation") {
        var result = ""
        repeat(100) {
            result += "item$it"
        }
    }
    
    // Benchmark StringBuilder (faster)
    benchmark.benchmarkBlock("StringBuilder") {
        val builder = StringBuilder()
        repeat(100) {
            builder.append("item$it")
        }
    }
    
    // Benchmark coroutines
    benchmark.benchmarkBlock("Coroutine launch") {
        val jobs = listOf(1..10).map { 
            launch { delay(1) }
        }
        jobs.joinAll()
    }
}
```

### Memory optimization:

Создайте `shared/src/commonMain/kotlin/com/skillsync/performance/MemoryOptimizations.kt`:

```kotlin
package com.skillsync.performance

import kotlinx.coroutines.flow.*

// 1. Use value classes for primitive wrappers (zero overhead)
@JvmInline
value class UserId(val value: String)

@JvmInline  
value class Price(val value: Double)

// 2. Use inline classes for collections (avoid allocations)
inline class UserList internal constructor(private val list: List<User>) {
    operator fun get(index: Int) = list[index]
    val size: Int get() = list.size
    
    companion object {
        fun from(list: List<User>) = UserList(list)
    }
}

// 3. Optimize Flow usage (avoid unnecessary collections)
fun <T> Flow<T>.optimizedProcessing(): Flow<T> = this
    .flowOn(Dispatchers.Default) // Process на background thread
    .cacheIn(CoroutineScope(SupervisorJob())) // Cache для multiple subscribers
    .onStart { /* Pre-warming logic */ }

// 4. Use structured concurrency properly
suspend fun optimizedDataLoading(): List<DataItem> {
    return withContext(Dispatchers.IO) {
        // Все IO операции в одном context
        loadFromDatabase() + loadFromCache()
    }
}

// 5. Avoid memory leaks с WeakReference
class CachedData<T>(private val data: T) {
    // Для больших объектов используйте WeakReference
}

// 6. Object pooling для frequently created objects
class ObjectPool<T : Any> (
    private val factory: () -> T,
    private val maxSize: Int = 100
) {
    private val pool = ArrayDeque<T>()
    
    fun acquire(): T {
        synchronized(pool) {
            return if (pool.isNotEmpty()) {
                pool.removeLast()
            } else {
                factory()
            }
        }
    }
    
    fun release(obj: T) {
        synchronized(pool) {
            if (pool.size < maxSize) {
                pool.addLast(obj)
            }
        }
    }
}

// 7. Lazy initialization для expensive objects
object ExpensiveService {
    private val instance by lazy { 
        println("Initializing expensive service...")
        ExpensiveServiceImpl() // Дорогая инициализация
    }
    
    fun getService() = instance
}

// 8. Use sealed classes вместо inheritance для better optimization
sealed class ResourceState<out T> {
    object Loading : ResourceState<Nothing>()
    data class Success<out T>(val data: T) : ResourceState<T>()
    data class Error(val exception: Exception) : ResourceState<Nothing>()
}

// 9. Optimize serialization (avoid reflection)
@Serializable
data class OptimizedDataClass(
    val id: Long,
    val name: String,
    // Вместо List<String> используйте Array<String> для performance
    val tags: Array<String> = emptyArray() 
)

// 10. Batch operations вместо individual calls
suspend fun batchInsert(items: List<Item>): Result<Unit> {
    return try {
        // Один вызов вместо N отдельных
        database.insertAll(items)
        Result.success(Unit)
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

---

## 📝 Домашнее задание (Модуль 1)

### Задача: Оптимизация и benchmarking KMP приложения

**Требования:**

1. **Expect/Actual Patterns:**
   - Реализуйте expect/actual для PlatformStorage (SharedPreferences / UserDefaults)
   - Создайте expect sealed class для platform-specific типов
   - Напишите unit-тесты для каждой платформы

2. **Custom Build Configuration:**
   - Настройте multi-target build (Android, iOS, macOS)
   - Добавьте product flavors (dev/staging/prod)
   - Создайте custom task для cross-compilation

3. **Performance Optimization:**
   - Напишите benchmarks для критических операций
   - Оптимизируйте memory usage (value classes, object pooling)
   - Улучшите startup time на 20%+

4. **Documentation:**
   - Создайте PERFORMANCE.md с benchmark results
   - Документируйте все оптимизации

**Критерии сдачи:**
- ✅ Expect/actual работает на всех платформах
- ✅ Multi-target build успешно компилируется
- ✅ Benchmarks показывают улучшение performance
- ✅ Memory usage снижен на 15%+

---

**Следующий модуль:** В Module_02 мы изучим advanced UI patterns с Compose Multiplatform и SwiftUI.

Удачи! 🚀

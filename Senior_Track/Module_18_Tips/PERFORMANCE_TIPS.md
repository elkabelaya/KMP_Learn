# ⚡ Performance Tips

**Optimize your KMP apps - from build speed к runtime performance**

---

## 📋 Table о Contents
- [Build Performance](#build-performance)
- [Runtime Performance](#runtime-performance)  
- [Memory Optimization](#memory-optimization)
- [Network Performance](#network-performance)
- [UI Performance](#ui-performance)
- [Profiling Tools](#profiling-tools)

---

## Build Performance

### Gradle Optimization

#### 1. Increase JVM Memory
**Problem:** Out о memory errors during builds  
**Solution:**

```kotlin
// gradle.properties
org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=512m
```

**Impact:** 30-50% faster builds для large projects

#### 2. Enable Build Cache
**Problem:** Rebuilding unchanged modules  
**Solution:**

```kotlin
// settings.gradle.kts
buildCache {
    local {
        enabled = true
    }
    
    // Optional: Remote cache для teams
    remote<HttpBuildCache> {
        url = uri("https://your-cache-server.com/cache")
        isPush = true
    }
}
```

**Impact:** 40-60% faster incremental builds

#### 3. Use Configuration Cache
**Problem:** Gradle reconfigures project every build  
**Solution:**

```bash
# Command line
./gradlew build --configuration-cache

# Or в gradle.properties
org.gradle.configuration-cache=true
```

**Impact:** 20-40% faster builds

#### 4. Parallel Builds
**Problem:** Modules build sequentially  
**Solution:**

```kotlin
// gradle.properties
org.gradle.parallel=true
org.gradle.workers.max=4  # Adjust based на CPU cores
```

**Impact:** 25-35% faster builds

### Kotlin Compiler Optimization

#### 1. Free Compilation
**Problem:** Recompile entire modules  
**Solution:** Enable incremental compilation (default в Kotlin 1.9+)

```kotlin
// build.gradle.kts
kotlin {
    sourceSets {
        commonMain {
            kotlin.srcDir("src/commonMain/kotlin")
        }
    }
}
```

#### 2. Avoid Full Rebuilds
**Tips:**
- Build only what you need: `./gradlew :shared:build`
- Use `--continue` к build all modules even if one fails
- Exclude tests during development: `./gradlew assembleDebug -x test`

### Platform-Specific Optimizations

#### Android:
```kotlin
// build.gradle.kts (android)
android {
    // Enable R8 full mode
    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles += defaultProguardFile("proguard-android-optimize.txt")
        }
    }
    
    // Reduce build variants
    flavorDimensions += "version"
    productFlavors {
        create("production")  // Don't build debug during release
    }
}
```

#### iOS:
```bash
# Use incremental builds в Xcode
# Product → Build (not Clean Build)

# Archive only when needed
# Product → Archive (not for every test)
```

---

## Runtime Performance

### Coroutines Optimization

#### 1. Use Appropriate Dispatchers
```kotlin
// ❌ Bad: Using Default для I/O
val result = withContext(Dispatchers.Default) {
    api.fetchData()  // I/O operation!
}

// ✅ Good: Use IO dispatcher для I/O  
val result = withContext(Dispatchers.IO) {
    api.fetchData()
}

// ✅ Good: Use Default для CPU-intensive work
val result = withContext(Dispatchers.Default) {
    processLargeDataset()  // CPU operation
}

// ✅ Good: Use Main для UI updates (Compose does this automatically)
LaunchedEffect(key1) {
    updateUI()  // Runs на Main dispatcher
}
```

#### 2. Avoid Coroutine Leaks
```kotlin
// ❌ Bad: Unbounded coroutine creation
Button(onClick = {
    viewModel.loadData()  // Creates new coroutine every click!
})

// ✅ Good: Use existing scope  
@Composable
fun Screen(viewModel: ViewModel = hiltViewModel()) {
    LaunchedEffect(Unit) {
        viewModel.data.collect { /* handle */ }
    }
}

// ✅ Better: Use derivedStateOf для expensive computations
val processedData = remember {
    derivedStateOf { viewModel.rawData ->
        expensiveComputation(viewModel.rawData)
    }
}
```

#### 3. Cancel Unneeded Work
```kotlin
// ✅ Good: Cancel when navigating away  
@Composable
fun Screen() {
    val scope = rememberCoroutineScope()
    
    LaunchedEffect(Unit) {
        scope.launch {
            loadData().collect { data ->
                // Handle data
            }
        }
    }
    
    // Coroutine automatically cancels when composable recomposes
}
```

### Flow Optimization

#### 1. Use Cold vs Hot Flows Appropriately
```kotlin
// ✅ Cold Flow (new subscription = new execution)
val searchResults: Flow<Result> = flow {
    emit(api.search(query))  // Runs every time collected
}

// ✅ Hot Flow (shared subscription)  
val locationUpdates = callbackFlow {
    locationListener.addListener { location ->
        trySend(location)
    }
}.shareIn(viewModelScope, SharingStarted.WhileSubscribed(5000))
```

#### 2. Buffer Strategies
```kotlin
// ✅ Use buffer для high-frequency data
val sensorData = rawSensorFlow.buffer(DROP_LATEST)

// ✅ Use conflate для latest value only
val currentLocation = locationFlow.conflate()

// ✅ Use debounce для user input  
val searchQuery = searchTextFlow.debounce(300)
```

#### 3. Avoid Unnecessary Collectors
```kotlin
// ❌ Bad: Multiple collectors
flow.collect { /* work 1 */ }
flow.collect { /* work 2 */ }

// ✅ Good: Single collector with multiple operations
flow
    .map { transform(it) }
    .filter { shouldProcess(it) }
    .collect { handle(it) }
```

---

## Memory Optimization

### 1. Avoid Memory Leaks

#### ViewModel Best Practices:
```kotlin
// ✅ Good: Use ViewModelScope  
class MyViewModel : ViewModel() {
    private val _data = MutableStateFlow<Data?>(null)
    val data: StateFlow<Data?> = _data.asStateFlow()
    
    init {
        viewModelScope.launch {
            loadData().collect { _data.value = it }
        }
    }
}

// ❌ Bad: Using ViewModel as coroutine scope directly
class MyViewModel : ViewModel(), CoroutineScope by viewModelScope  // Don't do this!
```

#### Compose Best Practices:
```kotlin
// ✅ Good: Use remember к prevent recreation
@Composable
fun Screen() {
    val list = remember { expensiveListCreation() }
    
    // List created once, not on every recomposition
}

// ✅ Good: Use derivedStateOf для computed values  
val sortedList = remember(list) {
    derivedStateOf { list.sorted() }
}
```

### 2. Optimize Data Structures

#### Use Appropriate Collections:
```kotlin
// ✅ For read-only data: use immutable collections
val items: List<Item> = listOf(...)  // Not MutableList

// ✅ For large datasets: use Paging 3
val pager = remember {
    pagingFlow {
        load(initialLoadSize = 20) {
            database.getItems(offset = 0, limit = 20)
        }
    }.asPagingData()
}

// ✅ For caching: use LRU cache
val cache = MutableSortedMap<String, Data>(compareByDescending { it.timestamp })
```

### 3. Image Optimization

#### Compose Image Loading:
```kotlin
// ✅ Good: Use Coil с proper sizing
AsyncImage(
    model = imageUrl,
    contentDescription = description,
    contentScale = ContentScale.Crop,
    imageLoader = ImageLoader.Builder(context)
        .memoryCache { 
            MemoryCache.Builder(context)
                .maxSizePercent(0.25)  // 25% о RAM
                .build() 
        }
        .build()
)

// ❌ Bad: Loading full-resolution images
Image(painter = painterResource(id = R.drawable.hd_image), ...)
```

---

## Network Performance

### 1. Implement Smart Caching

```kotlin
class Repository(
    private val api: ApiService,
    private val cache: DataCache
) {
    suspend fun getData(id: String): Result<Data> {
        // Try cache first
        val cached = cache.get(id)
        if (cached != null && !isStale(cached)) {
            return Result.Success(cached)
        }
        
        // Fetch from network  
        val fresh = api.fetchData(id)
        cache.put(id, fresh)
        
        return Result.Success(fresh)
    }
    
    private fun isStale(data: Data): Boolean {
        return System.currentTimeMillis() - data.timestamp > CACHE_DURATION
    }
}
```

### 2. Request Optimization

#### Batch Requests:
```kotlin
// ❌ Bad: Multiple sequential requests
suspend fun loadAll() {
    val users = api.getUsers()
    val posts = api.getPosts()  
    val comments = api.getComments()
}

// ✅ Good: Parallel requests
suspend fun loadAll() {
    val users = async { api.getUsers() }
    val posts = async { api.getPosts() }
    val comments = async { api.getComments() }
    
    return users.await() to posts.await() to comments.await()
}

// ✅ Better: Single batched request  
suspend fun loadAll() {
    return api.getUsersPostsCommentsBatch()  // Single network call
}
```

#### Request Compression:
```kotlin
// OkHttp interceptor для compression
val client = OkHttpClient.Builder()
    .addInterceptor { chain ->
        val originalRequest = chain.request()
        
        val compressedRequest = originalRequest.newBuilder()
            .header("Accept-Encoding", "gzip, deflate")
            .build()
        
        chain.proceed(compressedRequest)
    }
    .build()
```

### 3. Connection Pooling

```kotlin
// Shared HTTP client с connection pooling
object HttpClient {
    val client = OkHttpClient.Builder()
        .connectionPool(ConnectionPool(
            maxIdleConnections = 5,
            keepAliveDuration = 5 minutes
        ))
        .build()
}

// Use same client для all requests
val api = Retrofit.Builder()
    .client(HttpClient.client)  // Reuses connections
    .build()
```

---

## UI Performance

### Compose Optimization

#### 1. Prevent Unnecessary Recompositions
```kotlin
// ✅ Use remember к stabilize references  
@Composable
fun Screen(viewModel: ViewModel) {
    val list = remember { viewModel.data.toList() }  // Stable reference
    
    LazyColumn {
        items(list) { item ->
            ItemRow(item)  // Won't recompose when parent does
        }
    }
}

// ✅ Use derivedStateOf для expensive computations  
val filteredList = remember(viewModel.rawData) {
    derivedStateOf { viewModel.rawData.filter { it.visible } }
}
```

#### 2. Optimize Lists
```kotlin
// ✅ Use LazyColumn/LazyRow для long lists  
LazyColumn {
    items(pagedData) { item ->
        ItemRow(item)  // Only visible items rendered
    }
}

// ✅ Use placeholder к prevent layout shifts  
AsyncImage(
    model = imageUrl,
    contentDescription = desc,
    placeholder = rememberAsyncImagePainter(imageVector = ImageLoader.Placeholder)
)

// ✅ Use item key к help recycling  
LazyColumn {
    items(items, key = { it.id }) { item ->  // Stable key
        ItemRow(item)
    }
}
```

#### 3. Animation Performance
```kotlin
// ✅ Use animate*AsState для simple animations  
val animatedAlpha = animateFloatAsTargetValueIfChanging(
    targetValue = isVisible.toFloat(),
    animationSpec = tween(durationMillis = 200)
)

// ❌ Avoid: Complex animations на main thread during scrolling  
// ✅ Use: Hardware-accelerated animations only
```

### Platform-Specific UI Tips

#### Android:
- Use `ViewBinding` instead о findViewById (if using Views)
- Enable hardware acceleration: `android:hardwareAccelerated="true"`
- Use ConstraintLayout instead о nested LinearLayouts

#### iOS:
- Use `UIView.animate` для simple animations  
- Avoid complex calculations в `layoutSubviews`
- Use `UICollectionView` instead о UITableView для complex layouts

---

## Profiling Tools

### Android Profiling

#### 1. Android Studio Profiler
```
View → Tool Windows → Profiler

Features:
- CPU usage (method tracing)
- Memory allocation  
- Network traffic
- Battery usage
```

#### 2. Systrace для Animation Issues
```bash
# Capture 10 seconds о UI thread activity
adb shell systrace --time=10 --buffer_size=2048 gfx view input

# Analyze в Chrome: chrome://tracing
```

### iOS Profiling

#### 1. Instruments (Xcode)
```
Product → Profile

Instruments to use:
- Time Profiler: CPU hotspots  
- Allocations: Memory leaks
- Energy Log: Battery impact
- Network: API calls
```

#### 2. Xcode Performance Reports
```
Product → Analyze → Report Performance Issues

Checks for:
- Slow UI updates  
- Memory leaks
- Excessive energy usage
```

### Multiplatform Profiling

#### 1. Shared Code Profiling
```kotlin
// Add timing instrumentation в shared code  
class PerformanceMonitor {
    fun <T> measure(name: String, block: () -> T): T {
        val start = System.currentTimeMillis()
        val result = block()
        val duration = System.currentTimeMillis() - start
        
        Log.d("PERF", "$name took $duration ms")
        return result
    }
}

// Usage:
val data = performanceMonitor.measure("loadData") {
    repository.getData()
}
```

#### 2. Custom Metrics Collection
```kotlin
// Collect performance metrics  
data class PerformanceMetrics(
    val startupTime: Long,
    val firstFrameTime: Long,  
    val apiLatency: List<Long>,
    val memoryUsage: Long
)

class PerformanceTracker {
    private val metrics = mutableListOf<PerformanceMetrics>()
    
    fun record(metrics: PerformanceMetrics) {
        metrics.add(metrics)
        
        // Send to analytics (if average > threshold)
        if (metrics.averageStartupTime() > 3000) {
            analytics.track("slow_startup", metrics)
        }
    }
}
```

---

## Quick Checklist

### Build Performance:
- [ ] Enable Gradle build cache  
- [ ] Use configuration cache
- [ ] Increase JVM memory (4GB+)
- [ ] Build only needed modules

### Runtime Performance:  
- [ ] Use correct coroutine dispatchers
- [ ] Avoid memory leaks (ViewModelScope)
- [ ] Implement smart caching  
- [ ] Optimize network requests

### UI Performance:
- [ ] Use LazyColumn/LazyRow для lists  
- [ ] Stabilize composable references with remember
- [ ] Use derivedStateOf для expensive computations  
- [ ] Profile animations

### Memory:
- [ ] Monitor memory usage в profiler
- [ ] Implement proper caching strategies  
- [ ] Optimize image loading
- [ ] Avoid holding references unnecessarily

---

## Resources

### Official Documentation:
- [Kotlin Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)
- [Compose Performance](https://developer.android.com/jetpack/compose/performance)  
- [Android Performance](https://developer.android.com/topic/performance)
- [iOS Performance](https://developer.apple.com/videos/search/performance/)

### Tools:
- **Android Studio Profiler:** Built-in  
- **Xcode Instruments:** Built-in
- **Perfetto:** Advanced Android tracing
- **Flipper:** Cross-platform debugging

### Books:
- "High Performance Android" by Alex Newman  
- "iOS Performance Optimization" by various authors

---

**Remember:** Measure before optimizing! Profile first, then optimize based на data. 📊

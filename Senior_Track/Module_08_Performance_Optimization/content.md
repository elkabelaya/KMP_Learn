# 📘 Модуль 5: Оптимизация производительности

**Добро пожаловать в пятый модуль Senior Track!**  
В этом модуле вы научитесь профилировать приложения, находить узкие места и оптимизировать производительность на всех уровнях: от запуска до рендеринга UI. Вы создадите инструмент для мониторинга метрик в реальном времени.

**Цели модуля:**
1. Освоить профилирование на Android (Profiler, Systrace) и iOS (Instruments)
2. Научиться оптимизировать время запуска приложения (Cold/Warm/Hot start)
3. Оптимизировать использование памяти и избежать memory leaks
4. Ускорить рендеринг UI (Compose оптимизации)
5. Создать встроенный Performance Monitor для отслеживания метрик

**Время выполнения:** ~30–40 часов (6 недель).

---

## 1. Введение: Метрики производительности

### Ключевые метрики (Google Core Vitals):

#### ⏱️ Time to Interactive (TTI)
Время от нажатия на иконку до возможности взаимодействия с приложением.

**Целевые значения:**
- Cold Start: < 3 секунды (Android), < 2 секунды (iOS)
- Warm Start: < 1.5 секунды

#### 🎨 First Contentful Paint (FCP)
Время до отображения первого контента.

**Целевые значения:** < 1.8 секунды

#### 📉 Memory Usage
Потребление памяти приложения.

**Целевые значения:**
- Android: < 150MB (средний класс), < 200MB (флагманы)
- iOS: < 100MB

#### 🔋 Battery Impact
Влияние на заряд батареи.

**Целевые значения:** < 5% в час активного использования

#### 📶 Network Efficiency
Эффективность использования сети.

**Целевые значения:** Минимизация запросов, использование кэша

---

## 2. Теория: Профилирование на Android

### Инструменты профилирования:

#### 📊 Android Studio Profiler
Встроенный инструмент для мониторинга CPU, Memory, Network и Energy.

**Как использовать:**
1. Запустите приложение в Debug режиме
2. Откройте Profiler (View → Tool Windows → Profiler)
3. Выберите устройство и начните запись

#### 🔍 Memory Profiler
Находит memory leaks и оптимизирует использование памяти.

**Типичные проблемы:**
- **Memory Leaks:** Объекты не освобождаются после завершения работы
- **Fragmentation:** Разрозненное выделение памяти
- **Over-allocation:** Чрезмерное создание объектов

**Как найти leak:**
1. Сделайте Heap Dump в Profiler
2. Найдите объекты с большим количеством ссылок
3. Проверьте Reference Tree на циклические ссылки

#### ⚡ CPU Profiler
Находит медленные методы и оптимизирует вычисления.

**Как использовать:**
1. Запустите CPU Profiling
2. Выполните медленную операцию (например, прокрутка списка)
3. Проанализируйте Flame Graph

#### 📡 Network Profiler
Анализирует сетевые запросы.

**Что искать:**
- Излишние запросы (N+1 проблема)
- Большие ответы (> 500KB)
- Медленные запросы (> 1 секунда)

#### 🔋 Energy Profiler
Показывает влияние на батарею.

**Что искать:**
- Частые wakeups (AlarmManager, WorkManager)
- Активное использование GPS
- Постоянные сетевые соединения

---

## 3. Теория: Профилирование на iOS

### Инструменты профилирования (Instruments):

#### 📊 Time Profiler
Анализ использования CPU по методам.

**Как использовать:**
1. Xcode → Product → Profile
2. Выберите шаблон "Time Profiler"
3. Запустите и выполните медленные операции

#### 💾 Allocations & Leaks
Находит утечки памяти и анализирует выделение.

**Как использовать:**
1. Выберите шаблон "Allocations"
2. Включите "Track allocations over time"
3. Ищите объекты, которые не освобождаются

#### ⚡ Energy Log
Анализирует влияние на батарею.

**Что искать:**
- High CPU usage
- Network activity
- Screen on time

#### 🎨 Core Animation
Анализирует производительность UI.

**Что искать:**
- Jank (кадры < 60 FPS)
- Off-screen rendering
- Excessive layout passes

---

## 4. Практика: Оптимизация времени запуска (Cold Start)

### Проблема
Приложение долго запускается из-за:
- Инициализации зависимостей
- Загрузки данных с сервера
- Рендеринга стартового экрана

### Решение: Lazy Initialization + Preloading

#### Шаг 1: Анализ времени запуска

Создайте `shared/src/commonMain/kotlin/performance/StartupProfiler.kt`:

```kotlin
package performance

import kotlinx.datetime.Instant

class StartupProfiler {
    private val events = mutableListOf<StartupEvent>()
    private var startTime: Instant? = null
    
    fun start() {
        startTime = Instant.now()
        logEvent("Startup started")
    }
    
    fun mark(event: String) {
        logEvent(event)
    }
    
    private fun logEvent(event: String) {
        val currentTime = Instant.now()
        val elapsed = startTime?.let { 
            (currentTime.epochSeconds - it.epochSeconds) * 1000 + 
            (currentTime.nanoseconds - it.nanoseconds) / 1_000_000 
        } ?: 0L
        
        events.add(StartupEvent(event, elapsed))
    }
    
    fun getReport(): List<StartupEvent> = events
    
    data class StartupEvent(
        val name: String,
        val elapsedMs: Long
    )
}

// Глобальный инстанс
val startupProfiler = StartupProfiler()
```

#### Шаг 2: Оптимизация инициализации зависимостей

Создайте `shared/src/commonMain/kotlin/di/LazyContainer.kt`:

```kotlin
package di

import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock

class LazyContainer<T>(private val factory: () -> T) {
    private var value: T? = null
    private val mutex = Mutex()
    
    suspend fun get(): T {
        return value ?: mutex.withLock {
            value ?: factory().also { value = it }
        }
    }
}

// Использование в DI контейнере
class AppContainer {
    // Lazy инициализация - создается только при первом использовании
    val database = LazyContainer { 
        startupProfiler.mark("Database initialization started")
        createDatabase()
        startupProfiler.mark("Database initialized")
    }
    
    val networkClient = LazyContainer { 
        startupProfiler.mark("Network client initialization started")
        createNetworkClient()
        startupProfiler.mark("Network client initialized")
    }
    
    val repository = LazyContainer { 
        startupProfiler.mark("Repository initialization started")
        createRepository(database.get(), networkClient.get())
        startupProfiler.mark("Repository initialized")
    }
}

// В MainActivity/MainViewController
fun main() {
    startupProfiler.start()
    
    // Инициализируем только то, что нужно для первого экрана
    val container = AppContainer()
    
    // Остальное инициализируем лениво
}
```

#### Шаг 3: Preloading данных для первого экрана

Создайте `shared/src/commonMain/kotlin/performance/DataPreloader.kt`:

```kotlin
package performance

import kotlinx.coroutines.*

class DataPreloader(
    private val repository: Repository,
    private val startupProfiler: StartupProfiler
) {
    // Предзагружаем данные для первого экрана в background
    suspend fun preloadFirstScreenData() {
        startupProfiler.mark("Preloading first screen data")
        
        // Асинхронная предзагрузка (не блокирует запуск)
        CoroutineScope(Dispatchers.IO).launch {
            try {
                // Загружаем только критичные данные
                repository.getInitialData()
                
                startupProfiler.mark("First screen data preloaded")
            } catch (e: Exception) {
                // Игнорируем ошибки предзагрузки - приложение должно работать
            }
        }
    }
}

// Использование при запуске
fun main() {
    startupProfiler.start()
    
    val container = AppContainer()
    val preloader = DataPreloader(container.repository.get(), startupProfiler)
    
    // Запускаем предзагрузку в background
    preloader.preloadFirstScreenData()
    
    // Показываем стартовый экран сразу
    showSplashScreen()
}
```

#### Шаг 4: Оптимизация Compose при запуске

Создайте `shared/src/commonMain/kotlin/ui/SplashScreen.kt`:

```kotlin
package ui

import androidx.compose.animation.core.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier

@Composable
fun SplashScreen(
    onReady: () -> Unit,
    modifier: Modifier = Modifier
) {
    var showMainScreen by remember { mutableStateOf(false) }
    
    // Анимация логотипа
    val alpha = remember {
        animateFloatAsState(
            targetValue = if (showMainScreen) 0f else 1f,
            animationSpec = tween(500)
        )
    }
    
    LaunchedEffect(Unit) {
        // Ждем пока критичные данные загрузятся (максимум 2 секунды)
        delay(2000)
        showMainScreen = true
        
        // Переход на главный экран через 500ms после анимации
        delay(500)
        onReady()
    }
    
    // Показываем логотип с анимацией
    AnimatedVisibility(
        visible = !showMainScreen,
        modifier = modifier
    ) {
        LogoAnimation(alpha.value)
    }
}

@Composable
fun LogoAnimation(alpha: Float) {
    // Оптимизированная анимация логотипа
    // Используем векторные иконки вместо растровых изображений
}
```

---

## 5. Практика: Оптимизация памяти и борьба с memory leaks

### Типичные причины memory leaks в KMP:

#### 1️⃣ Забытые корутины
```kotlin
// ПЛОХО: Корутин не отменяется при уничтожении экрана
@Composable
fun LeakyScreen() {
    LaunchedEffect(Unit) { // ❌ Нет отмены при уничтожении
        while (true) {
            delay(1000)
            // Бесконечный цикл
        }
    }
}

// ХОРОШО: Корутин отменяется автоматически
@Composable
fun OptimizedScreen() {
    LaunchedEffect(Unit) { // ✅ Отменяется при уничтожении Composable
        // Работает только пока экран активен
    }
}
```

#### 2️⃣ Глобальные коллекции без очистки
```kotlin
// ПЛОХО: Коллекция растет бесконечно
object GlobalCache {
    val items = mutableListOf<Item>() // ❌ Никогда не очищается
    
    fun addItem(item: Item) {
        items.add(item)
    }
}

// ХОРОШО: LRU Cache с ограниченным размером
class LruCache<T>(private val maxSize: Int = 100) {
    private val cache = LinkedHashMap<T, Any>(maxSize, 0.75f, true)
    
    @Synchronized
    fun get(key: T): Any? {
        return cache[key]
    }
    
    @Synchronized
    fun put(key: T, value: Any) {
        cache[key] = value
        
        // Удаляем старые элементы если превышен лимит
        if (cache.size > maxSize) {
            cache.keys.firstOrNull()?.let { cache.remove(it) }
        }
    }
}
```

#### 3️⃣ Слушатели без отписки
```kotlin
// ПЛОХО: Слушатель не удаляется
class LeakyObserver {
    init {
        eventBus.subscribe(this) // ❌ Никогда не отписываемся
    }
    
    fun onEvent(event: Event) { /* ... */ }
}

// ХОРОШО: Слушатель удаляется при уничтожении
class OptimizedObserver : Disposable {
    private val subscription = eventBus.subscribe(this)
    
    fun onEvent(event: Event) { /* ... */ }
    
    override fun dispose() {
        subscription.cancel() // ✅ Отписываемся
    }
}
```

### Шаг 1: Создание Memory Monitor

Создайте `shared/src/commonMain/kotlin/performance/MemoryMonitor.kt`:

```kotlin
package performance

import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

class MemoryMonitor {
    private val _memoryUsage = MutableStateFlow(0L)
    val memoryUsage: StateFlow<Long> = _memoryUsage.asStateFlow()
    
    private var monitorJob: Job? = null
    
    fun startMonitoring(periodMs: Long = 1000) {
        monitorJob = CoroutineScope(Dispatchers.Default).launch {
            while (isActive) {
                try {
                    val usage = getMemoryUsage()
                    _memoryUsage.value = usage
                    
                    // Логгируем если память превышает лимит
                    if (usage > MEMORY_LIMIT_MB * 1024 * 1024) {
                        logWarning("Memory usage high: ${usage / (1024 * 1024)} MB")
                    }
                } catch (e: Exception) {
                    // Игнорируем ошибки мониторинга
                }
                
                delay(periodMs)
            }
        }
    }
    
    fun stopMonitoring() {
        monitorJob?.cancel()
        monitorJob = null
    }
    
    private fun getMemoryUsage(): Long {
        // Платформозависимая реализация
        return getPlatformMemoryUsage()
    }
    
    companion object {
        const val MEMORY_LIMIT_MB = 150
        
        expect fun getPlatformMemoryUsage(): Long
    }
}

// Реализация для Android
actual fun getPlatformMemoryUsage(): Long {
    val runtime = Runtime.getRuntime()
    return (runtime.totalMemory() - runtime.freeMemory()) / 1024 / 1024
}

// Реализация для iOS
actual fun getPlatformMemoryUsage(): Long {
    // Используем нативный API для получения использования памяти
    return 0L // Заглушка - реальная реализация требует C-interop
}
```

### Шаг 2: Автоматическая очистка кэша при нехватке памяти

Создайте `shared/src/commonMain/kotlin/performance/CacheManager.kt`:

```kotlin
package performance

class CacheManager(
    private val memoryMonitor: MemoryMonitor,
    private val caches: List<ClearableCache>
) {
    init {
        // Мониторим память и очищаем кэш при необходимости
        memoryMonitor.memoryUsage.collect { usage ->
            if (usage > MEMORY_LIMIT_MB * 1024 * 1024 * 0.8) {
                // Очистка кэша если память > 80% от лимита
                clearCaches()
            }
        }
    }
    
    private fun clearCaches() {
        caches.forEach { it.clear() }
        logInfo("Caches cleared due to high memory usage")
    }
}

interface ClearableCache {
    fun clear()
}

// Пример реализации LRU Cache с очисткой
class LruImageCache : ClearableCache {
    private val cache = LruCache<String, ByteArray>(100)
    
    override fun clear() {
        cache.evictAll()
    }
}
```

---

## 6. Практика: Создание встроенного Performance Monitor

### Шаг 1: UI компонент для отображения метрик

Создайте `shared/src/commonMain/kotlin/ui/components/PerformanceMonitor.kt`:

```kotlin
package ui.components

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import performance.MemoryMonitor

@Composable
fun PerformanceMonitor(
    memoryMonitor: MemoryMonitor,
    modifier: Modifier = Modifier,
    enabled: Boolean = true
) {
    if (!enabled) return
    
    val memoryUsage by memoryMonitor.memoryUsage.collectAsState()
    
    Card(
        modifier = modifier
            .fillMaxWidth()
            .padding(8.dp),
        colors = CardDefaults.cardColors(
            containerColor = Color(0xFF1A237E)
        )
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                text = "Performance Monitor",
                style = MaterialTheme.typography.titleMedium,
                color = Color.White
            )
            
            Spacer(modifier = Modifier.height(8.dp))
            
            // Memory usage
            Row(verticalAlignment = Alignment.CenterVertically) {
                Text(
                    text = "Memory: ",
                    color = Color.White,
                    style = MaterialTheme.typography.bodyMedium
                )
                
                val memoryMB = (memoryUsage / 1024 / 1024).toInt()
                Text(
                    text = "$memoryMB MB",
                    color = if (memoryMB > 150) Color.Red else Color.Green,
                    style = MaterialTheme.typography.bodyMedium
                )
            }
            
            // FPS counter (будет добавлен позже)
            Row(verticalAlignment = Alignment.CenterVertically) {
                Text(
                    text = "FPS: ",
                    color = Color.White,
                    style = MaterialTheme.typography.bodyMedium
                )
                
                Text(
                    text = "60", // Заглушка - будет реальная метрика
                    color = Color.Green,
                    style = MaterialTheme.typography.bodyMedium
                )
            }
            
            // Startup time
            Row(verticalAlignment = Alignment.CenterVertically) {
                Text(
                    text = "Startup: ",
                    color = Color.White,
                    style = MaterialTheme.typography.bodyMedium
                )
                
                Text(
                    text = "1.2s", // Заглушка - будет реальная метрика
                    color = Color.Green,
                    style = MaterialTheme.typography.bodyMedium
                )
            }
        }
    }
}
```

### Шаг 2: Интеграция в приложение

Создайте `shared/src/commonMain/kotlin/ui/App.kt`:

```kotlin
package ui

import androidx.compose.runtime.*
import performance.MemoryMonitor
import ui.components.PerformanceMonitor

@Composable
fun App(
    memoryMonitor: MemoryMonitor,
    debugMode: Boolean = false
) {
    // Запускаем мониторинг в debug режиме
    LaunchedEffect(debugMode) {
        if (debugMode) {
            memoryMonitor.startMonitoring()
        } else {
            memoryMonitor.stopMonitoring()
        }
    }
    
    MaterialTheme {
        Column(
            modifier = Modifier.fillMaxSize()
        ) {
            // Основной контент приложения
            AppNavHost()
            
            // Performance Monitor (только в debug режиме)
            if (debugMode) {
                PerformanceMonitor(
                    memoryMonitor = memoryMonitor,
                    enabled = true
                )
            }
        }
    }
}
```

---

## 📝 Домашнее задание (Модуль 5)

### Задача: Оптимизировать производительность SkillSync

**Требования:**

1. **Профилирование:**
   - Запрофилируйте приложение на Android (Android Studio Profiler) и iOS (Instruments)
   - Найдите минимум 3 узких места в производительности
   - Создайте отчет с метриками "до" и "после"

2. **Оптимизация запуска:**
   - Уменьшите Cold Start время до < 2 секунд (Android), < 1.5 секунды (iOS)
   - Реализуйте lazy initialization для тяжелых зависимостей
   - Добавьте предзагрузку данных для первого экрана

3. **Оптимизация памяти:**
   - Найдите и исправьте memory leaks (используйте Heap Dump)
   - Реализуйте LRU Cache для изображений с ограничением по размеру
   - Добавьте автоматическую очистку кэша при нехватке памяти

4. **Оптимизация UI:**
   - Устраните jank в списках (целевая метрика: 60 FPS при прокрутке)
   - Оптимизируйте перерисовки Compose (используйте remember, derivedStateOf)
   - Добавьте LazyColumn для больших списков

5. **Performance Monitor:**
   - Создайте встроенный Performance Monitor (как в примере)
   - Отображайте: Memory Usage, FPS, Startup Time
   - Показывайте только в debug режиме

**Критерии сдачи:**
- ✅ Cold Start < 2s (Android), < 1.5s (iOS)
- ✅ Memory usage < 150MB в обычном режиме
- ✅ 60 FPS при прокрутке списков с 10k+ элементов
- ✅ Нет memory leaks (проверьте через Heap Dump)
- ✅ Performance Monitor отображает метрики в реальном времени

**Бонусные задания:**
- Добавьте трассировку сетевых запросов с таймингами
- Реализуйте предиктивную загрузку данных (predictive prefetching)
- Создайте кастомный sampler для профилирования бизнес-логики

---

## 💡 Советы по выполнению

1. **Измеряйте до и после:** Без метрик нельзя оценить эффективность оптимизаций.
2. **Оптимизируйте по-настоящему медленное:** Не тратьте время на микро-оптимизации.
3. **Используйте правильные инструменты:** Android Profiler для Android, Instruments для iOS.
4. **Тестируйте на реальном устройстве:** Эмуляторы могут скрывать проблемы с производительностью.
5. **Не жертвуйте читаемостью кода:** Оптимизация не должна делать код непонятным.

---

**Следующий модуль:** В Module_06_Testing_Quality мы изучим unit-тесты, integration-тесты, UI-тесты и настройку CI/CD пайплайнов.

Удачи! 🚀

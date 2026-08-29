# 📘 Модуль 8: Продвинутые темы

**Добро пожаловать в финальный модуль Senior Track!**  
В этом модуле вы изучите продвинутые темы KMP: C++ interop, машинное обучение, advanced concurrency patterns. Это уровень Senior+ разработчика KMP.

**Цели модуля:**
1. Освоить C++ interop для использования нативных библиотек в KMP
2. Интегрировать машинное обучение (TensorFlow Lite, Core ML) в KMP приложение
3. Изучить advanced concurrency: Actors, Channels, Flow operators
4. Настроить advanced build configuration и оптимизацию
5. Создать production-ready KMP приложение с best practices

**Время выполнения:** ~40–50 часов (8 недель).

---

## 1. C++ Interop: Использование нативных библиотек

### Зачем нужен C++ interop?

**Сценарии использования:**
- 📚 Использование существующих C/C++ библиотек (OpenCV, FFmpeg, SQLite)
- 🚀 Performance-critical код (обработка изображений, видео, шифрование)
- 🤖 Машинное обучение (TensorFlow C++, ONNX Runtime)
- 🔐 Криптография и безопасность

### Варианты interop:

#### 1️⃣ C Interop (FFI) - Простой
```kotlin
// Для чистого C кода
external fun printf(format: CString, vararg args: AnyVarArg)
```

#### 2️⃣ C++ Interop (Konan) - Продвинутый
```kotlin
// Для C++ библиотек с классами, наследованием и т.д.
val imageProcessor = ImageProcessor()
imageProcessor.process(image)
```

#### 3️⃣ Kotlin/Native C++ Interop - Полная поддержка
- Поддержка классов, наследования, шаблонов (с ограничениями)
- Автоматическая генерация bindings через C++ headers

---

### Практика: Интеграция OpenCV для обработки изображений

#### Шаг 1: Подготовка C++ библиотеки

Создайте `shared/src/commonMain/cpp/OpenCVWrapper/`:

```cpp
// OpenCVWrapper.h - Обертка для Kotlin
#pragma once

#include <string>
#include <vector>

// Упрощенная структура для передачи данных между C++ и Kotlin
struct ImageData {
    int width;
    int height;
    std::vector<unsigned char> pixels; // RGB data
};

class ImageProcessor {
public:
    ImageProcessor();
    ~ImageProcessor();
    
    // Обработка изображения (C++ API)
    ImageData processImage(ImageData input, int filterType);
    
    // Фильтры: 0 = None, 1 = Grayscale, 2 = Blur, 3 = Edge Detection
    int getFilterCount();
    std::string getFilterName(int filterIndex);
};

// C-style API для простого interop (альтернатива)
extern "C" {
    void* createImageProcessor();
    void destroyImageProcessor(void* processor);
    
    struct ImageData* processImageC(
        void* processor, 
        const unsigned char* pixels, 
        int width, 
        int height, 
        int filterType
    );
    
    void freeImageData(struct ImageData* data);
}
```

Создайте `shared/src/commonMain/cpp/OpenCVWrapper/ImageProcessor.cpp`:

```cpp
#include "OpenCVWrapper.h"
// #include <opencv2/opencv.hpp> // Раскомментируйте для реального OpenCV

ImageProcessor::ImageProcessor() {}

ImageProcessor::~ImageProcessor() {}

ImageData ImageProcessor::processImage(ImageData input, int filterType) {
    ImageData output = input;
    
    switch (filterType) {
        case 1: // Grayscale
            for (size_t i = 0; i < input.pixels.size(); i += 3) {
                unsigned char r = input.pixels[i];
                unsigned char g = input.pixels[i + 1];
                unsigned char b = input.pixels[i + 2];
                
                // Конвертация в grayscale
                unsigned char gray = (unsigned char)(0.299 * r + 0.587 * g + 0.114 * b);
                
                output.pixels[i] = gray;
                output.pixels[i + 1] = gray;
                output.pixels[i + 2] = gray;
            }
            break
            
        case 2: // Blur (упрощенный box blur)
            // Реализация blur фильтра...
            break
            
        case 3: // Edge Detection (Sobel)
            // Реализация edge detection...
            break
            
        default:
            // No filter - возвращаем как есть
            break;
    }
    
    return output;
}

int ImageProcessor::getFilterCount() {
    return 4; // None, Grayscale, Blur, Edge Detection
}

std::string ImageProcessor::getFilterName(int filterIndex) {
    switch (filterIndex) {
        case 0: return "None";
        case 1: return "Grayscale";
        case 2: return "Blur";
        case 3: return "Edge Detection";
        default: return "Unknown";
    }
}

// C-style API implementation
extern "C" {
    void* createImageProcessor() {
        return new ImageProcessor();
    }
    
    void destroyImageProcessor(void* processor) {
        delete static_cast<ImageProcessor*>(processor);
    }
    
    struct ImageData* processImageC(
        void* processor, 
        const unsigned char* pixels, 
        int width, 
        int height, 
        int filterType
    ) {
        ImageProcessor* proc = static_cast<ImageProcessor*>(processor);
        
        ImageData input;
        input.width = width;
        input.height = height;
        input.pixels.assign(pixels, pixels + width * height * 3);
        
        ImageData output = proc->processImage(input, filterType);
        
        // Конвертируем в C-style struct для возврата
        auto* result = new ImageData();
        *result = output;
        
        return result;
    }
    
    void freeImageData(struct ImageData* data) {
        delete data;
    }
}
```

#### Шаг 2: Настройка C++ interop в build.gradle.kts

Создайте `shared/build.gradle.kts`:

```kotlin
plugins {
    kotlin("multiplatform")
    id("com.android.library")
}

kotlin {
    androidTarget()
    
    iosX64()
    iosArm64()
    iosSimulatorArm64()
    
    // Настройка C++ interop
    applyDefaultHierarchyTemplate {
        common {
            group("native") {
                withAndroidTarget()
                withIosTargets()
            }
        }
    }
    
    sourceSets {
        val commonMain by getting {
            dependencies {
                // Kotlin/Native C++ interop автоматически подключается
            }
        }
        
        val nativeMain by creating {
            dependsOn(commonMain)
            
            // Указываем C++ источники
            kotlinOptions {
                freeCompilerArgs += listOf(
                    "-Xkotlin-native-dependencies",
                    "opencv:4.8.0" // Если используете OpenCV из package manager
                )
            }
        }
        
        val androidMain by getting {
            dependsOn(nativeMain)
        }
        
        val iosX64Main by creating {
            dependsOn(nativeMain)
        }
        
        val iosArm64Main by creating {
            dependsOn(nativeMain)
        }
        
        val iosSimulatorArm64Main by creating {
            dependsOn(nativeMain)
        }
    }
    
    // Настройка C++ компиляции
    compilerOptions {
        freeCompilerArgs += listOf(
            "-Xkotlin-native-dependencies",
            "-Xallow-no-source-files" // Разрешает компиляцию только C++ кода
        )
    }
}

android {
    namespace = "com.skillsync.shared"
    
    // Настройка C++ для Android (NDK)
    defaultConfig {
        externalNativeBuild {
            cmake {
                cppFlags += listOf("-std=c++17")
                arguments += "ARCH_ARM=x86"
            }
        }
    }
    
    buildTypes {
        getByName("release") {
            isMinifyEnabled = false
        }
    }
}

// CMakeLists.txt для Android
tasks.register<Exec>("generateCMake") {
    commandLine("echo", "CMake configuration for C++ interop")
}
```

Создайте `shared/CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.4.1)

project(SkillSyncNative)

# Включаем C++17
set(CMAKE_CXX_STANDARD 17)

# Добавляем наши C++ источники
add_library(
    skillsync_native
    SHARED
    src/commonMain/cpp/OpenCVWrapper/ImageProcessor.cpp
)

# Если используете OpenCV из AAR
# find_library(opencv_lib opencv_core)
# target_link_libraries(skillsync_native ${opencv_lib})

# Включаем logcat для отладки
find_loggable(loggable)
target_link_libraries(skillsync_native loggable)
```

#### Шаг 3: Kotlin обертка для C++ кода

Создайте `shared/src/nativeMain/kotlin/com/skillsync/natives/ImageProcessor.kt`:

```kotlin
package com.skillsync.natives

import platform.posix.*

// Автоматически сгенерированные классы из C++ headers
// (после компиляции будут доступны в platform/)

class ImageProcessorWrapper {
    private val processor: CPointer<ImageProcessor>
    
    init {
        // Создаем экземпляр C++ класса
        processor = createImageProcessor()
    }
    
    fun processImage(
        pixels: ByteArray, 
        width: Int, 
        height: Int, 
        filterType: Int
    ): ByteArray {
        // Вызываем C++ функцию через FFI
        val result = processImageC(
            processor, 
            pixels, 
            width, 
            height, 
            filterType
        )
        
        // Копируем данные из C++ в Kotlin
        val output = ByteArray(result.width * result.height * 3)
        memcopy(
            output.ptrTo(0), 
            result.pixels.data, 
            (result.width * result.height * 3).toULong()
        )
        
        // Освобождаем память C++
        freeImageData(result)
        
        return output
    }
    
    fun getFilterCount(): Int {
        // Вызов метода C++ класса (требует полной поддержки C++ interop)
        return 4 // Заглушка для примера
    }
    
    fun dispose() {
        destroyImageProcessor(processor)
    }
}

// Expect/Actual для платформенной специфики
expect fun createImageProcessor(): CPointer<ImageProcessor>
expect fun destroyImageProcessor(processor: CPointer<ImageProcessor>)
expect fun processImageC(
    processor: CPointer<ImageProcessor>,
    pixels: ByteArray,
    width: Int,
    height: Int,
    filterType: Int
): ImageData

expect fun freeImageData(data: ImageData)
```

Создайте `shared/src/androidMain/kotlin/com/skillsync/natives/ImageProcessor.android.kt`:

```kotlin
package com.skillsync.natives

import platform.posix.*

// Android реализация - используем JNI или нативную библиотеку
actual fun createImageProcessor(): CPointer<ImageProcessor> {
    // Загружаем нативную библиотеку
    System.loadLibrary("skillsync_native")
    
    // Вызываем C функцию из библиотеки
    return createImageProcessorNative()
}

actual fun destroyImageProcessor(processor: CPointer<ImageProcessor>) {
    destroyImageProcessorNative(processor)
}

actual fun processImageC(
    processor: CPointer<ImageProcessor>,
    pixels: ByteArray,
    width: Int,
    height: Int,
    filterType: Int
): ImageData {
    return processImageCNative(processor, pixels, width, height, filterType)
}

actual fun freeImageData(data: ImageData) {
    freeImageDataNative(data)
}

// Native функции (будут сгенерированы из C++ headers или объявлены вручную)
external fun createImageProcessorNative(): CPointer<ImageProcessor>
external fun destroyImageProcessorNative(processor: CPointer<ImageProcessor>)
external fun processImageCNative(
    processor: CPointer<ImageProcessor>,
    pixels: ByteArray,
    width: Int,
    height: Int,
    filterType: Int
): ImageData
external fun freeImageDataNative(data: ImageData)
```

Создайте `shared/src/iosMain/kotlin/com/skillsync/natives/ImageProcessor.ios.kt`:

```kotlin
package com.skillsync.natives

import platform.posix.*

// iOS реализация - используем C++ interop напрямую
actual fun createImageProcessor(): CPointer<ImageProcessor> {
    return createImageProcessor() // Автоматически сгенерировано из headers
}

actual fun destroyImageProcessor(processor: CPointer<ImageProcessor>) {
    destroyImageProcessor(processor) // Автоматически сгенерировано
}

actual fun processImageC(
    processor: CPointer<ImageProcessor>,
    pixels: ByteArray,
    width: Int,
    height: Int,
    filterType: Int
): ImageData {
    val result = processImageC(processor, pixels, width, height, filterType)
    return result
}

actual fun freeImageData(data: ImageData) {
    freeImageData(data) // Автоматически сгенерировано
}
```

---

## 2. Машинное обучение в KMP

### Варианты ML в KMP:

#### 1️⃣ TensorFlow Lite (Android + iOS)
```kotlin
// Android: TensorFlow Lite native library
// iOS: Core ML (конвертируем TF Lite модели)
```

#### 2️⃣ Core ML (iOS only) + Fallback для Android
```kotlin
// iOS: Native Core ML
// Android: TensorFlow Lite или серверный API
```

#### 3️⃣ Server-side ML (Ktor backend)
```kotlin
// Все ML на сервере, KMP приложение только отправляет запросы
```

### Практика: Интеграция TensorFlow Lite для распознавания изображений

#### Шаг 1: Подготовка ML модели

Создайте `shared/src/commonMain/resources/tflite/`:
- Загрузите `.tflite` модель (например, MobileNet для классификации изображений)

#### Шаг 2: ML Service с expect/actual

Создайте `shared/src/commonMain/kotlin/com/skillsync/ml/ImageClassifier.kt`:

```kotlin
package com.skillsync.ml

// Интерфейс для ML сервиса (100% KMP)
interface ImageClassifier {
    suspend fun classifyImage(imageData: ByteArray): List<ClassificationResult>
    
    fun isModelLoaded(): Boolean
    
    suspend fun loadModel(): Result<Unit>
}

data class ClassificationResult(
    val label: String,
    val confidence: Float // 0.0 - 1.0
)

// Expect/Actual для платформенной реализации
expect fun createImageClassifier(): ImageClassifier
```

Создайте `shared/src/androidMain/kotlin/com/skillsync/ml/ImageClassifier.android.kt`:

```kotlin
package com.skillsync.ml

import org.tensorflow.lite.Interpreter
import java.nio.ByteBuffer
import java.nio.ByteOrder

actual fun createImageClassifier(): ImageClassifier {
    return AndroidImageClassifier()
}

class AndroidImageClassifier : ImageClassifier {
    
    private var interpreter: Interpreter? = null
    private var isModelLoaded = false
    
    override suspend fun loadModel(): Result<Unit> {
        return try {
            // Загружаем модель из assets
            val model = loadModelFile("mobilenet_v1_1.0_224_quant.tflite")
            
            interpreter = Interpreter(model)
            isModelLoaded = true
            
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override fun isModelLoaded(): Boolean = isModelLoaded
    
    override suspend fun classifyImage(imageData: ByteArray): List<ClassificationResult> {
        require(isModelLoaded) { "Model not loaded" }
        
        // Преобразуем изображение в формат для TensorFlow Lite
        val inputBuffer = prepareInput(imageData)
        val outputBuffer = ByteBuffer.allocateFloat(1000) // 1000 классов
        
        // Запускаем инференс
        interpreter?.run(inputBuffer, outputBuffer)
        
        // Парсим результаты
        return parseResults(outputBuffer)
    }
    
    private fun prepareInput(imageData: ByteArray): ByteBuffer {
        // Нормализация и ресайз изображения до 224x224
        // Конвертация в float32 формат
        
        val inputBuffer = ByteBuffer.allocateDirect(1 * 224 * 224 * 3 * 4)
        inputBuffer.order(ByteOrder.nativeOrder())
        
        // Здесь должна быть логика преобразования изображения
        // Для упрощения - заглушка
        
        return inputBuffer
    }
    
    private fun parseResults(outputBuffer: ByteBuffer): List<ClassificationResult> {
        val results = mutableListOf<ClassificationResult>()
        
        // Читаем топ-5 результатов
        repeat(5) { index ->
            val confidence = outputBuffer.float
            
            // Получаем label из labels.txt (заглушка)
            val label = "Label_$index"
            
            results.add(ClassificationResult(label, confidence))
        }
        
        return results.sortedByDescending { it.confidence }
    }
    
    private fun loadModelFile(fileName: String): java.nio.MappedByteBuffer {
        // Загрузка из assets - требует Android Context
        // Реализация зависит от вашего приложения
        TODO("Implement model loading from assets")
    }
    
    override fun finalize() {
        interpreter?.close()
    }
}
```

Создайте `shared/src/iosMain/kotlin/com/skillsync/ml/ImageClassifier.ios.kt`:

```kotlin
package com.skillsync.ml

import platform.CoreML.*
import platform.Vision.*
import kotlinx.cinterop.*

actual fun createImageClassifier(): ImageClassifier {
    return IOSImageClassifier()
}

class IOSImageClassifier : ImageClassifier {
    
    private var model: VNCoreMLModel? = null
    private var isModelLoaded = false
    
    override suspend fun loadModel(): Result<Unit> {
        return try {
            // Загружаем Core ML модель из bundle
            val mlModel = MNImageClassifier(modelGroup: nil)
            model = VNCoreMLModel.create(from = mlModel)!!
            
            isModelLoaded = true
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override fun isModelLoaded(): Boolean = isModelLoaded
    
    override suspend fun classifyImage(imageData: ByteArray): List<ClassificationResult> {
        require(isModelLoaded) { "Model not loaded" }
        
        // Конвертируем ByteArray в CGImage
        val cgImage = imageData.toCGImage()
        
        // Создаем VNRequest для классификации
        val request = VNImageClassificationRequest(
            model!!, 
            count: 5, 
            completionHandler = { request, error ->
                // Обработка результатов в callback
            }
        )
        
        request.image = cgImage
        
        // Выполняем request
        val handler = VNImageRequestHandler(cgImage)
        
        try {
            handler.perform(arrayOf(request))
            
            // Получаем результаты
            val results = request.results as? NSArray 
                ?: return emptyList()
            
            return (0 until results.count).map { index ->
                val result = results.objectAtIndex(index) as! VNClassificationObjectObservation
                ClassificationResult(
                    label = result.identifier, 
                    confidence = result.confidence.toFloat()
                )
            }
        } catch (e: Exception) {
            throw e
        }
    }
}

// Extension для конвертации ByteArray в CGImage
fun ByteArray.toCGImage(): platform.CoreGraphics.CGImage {
    // Реализация конвертации...
    TODO("Implement ByteArray to CGImage conversion")
}
```

---

## 3. Advanced Concurrency Patterns

### Проблема: Race conditions и shared state

```kotlin
// ПЛОХО - race condition
class Counter {
    var count = 0
    
    fun increment() {
        count++ // Не атомарно!
    }
}

// ХОРОШО - Actor pattern
class CounterActor {
    private val mailbox = Channel<CounterCommand>(Channel.UNLIMITED)
    
    private var count = 0
    
    sealed class CounterCommand {
        object Increment : CounterCommand()
        data class GetCount(val replyChannel: Channel<Int>) : CounterCommand()
    }
    
    init {
        // Запускаем actor coroutine
        CoroutineScope(Dispatchers.Default).launch {
            processCommands()
        }
    }
    
    private suspend fun processCommands() {
        for (command in mailbox) {
            when (command) {
                is CounterCommand.Increment -> {
                    count++
                }
                is CounterCommand.GetCount -> {
                    command.replyChannel.send(count)
                }
            }
        }
    }
    
    fun increment() {
        mailbox.trySend(CounterCommand.Increment)
    }
    
    suspend fun getCount(): Int {
        val replyChannel = Channel<Int>(1)
        mailbox.send(CounterCommand.GetCount(replyChannel))
        return replyChannel.receive()
    }
}
```

### Pattern 1: Actor Model

Создайте `shared/src/commonMain/kotlin/com/skillsync/concurrency/Actor.kt`:

```kotlin
package com.skillsync.concurrency

import kotlinx.coroutines.*
import kotlinx.coroutines.channels.Channel

// Base Actor class
abstract class Actor<Command>(
    private val scope: CoroutineScope = CoroutineScope(SupervisorJob())
) {
    protected val mailbox: Channel<Command> = Channel(Channel.UNLIMITED)
    
    init {
        scope.launch {
            try {
                processCommands()
            } finally {
                mailbox.close()
            }
        }
    }
    
    protected abstract suspend fun processCommands()
    
    protected suspend fun receive(): Command {
        return mailbox.receive()
    }
    
    fun send(command: Command) {
        mailbox.trySend(command)
    }
    
    suspend fun sendAndWait(command: Command): Boolean {
        return mailbox.send(command)
    }
    
    fun dispose() {
        scope.cancel()
    }
}

// Example: Cache Actor
class CacheActor<K, V>(
    scope: CoroutineScope = CoroutineScope(SupervisorJob())
) : Actor<CacheCommand<K, V>>(scope) {
    
    private val cache = mutableMapOf<K, V>()
    
    sealed class CacheCommand<out K, out V> {
        data class Get(val key: K, val replyChannel: Channel<V?>) : CacheCommand<K, V>()
        data class Put(val key: K, val value: V) : CacheCommand<K, V>()
        data class Remove(val key: K) : CacheCommand<K, Nothing>()
        object Clear : CacheCommand<Nothing, Nothing>()
    }
    
    override suspend fun processCommands() {
        for (command in mailbox) {
            when (command) {
                is CacheCommand.Get -> {
                    val value = cache[command.key]
                    command.replyChannel.trySend(value)
                }
                is CacheCommand.Put -> {
                    cache[command.key] = command.value
                }
                is CacheCommand.Remove -> {
                    cache.remove(command.key)
                }
                is CacheCommand.Clear -> {
                    cache.clear()
                }
            }
        }
    }
    
    suspend fun get(key: K): V? {
        val replyChannel = Channel<V?>(1)
        send(CacheCommand.Get(key, replyChannel))
        return replyChannel.receive()
    }
    
    fun put(key: K, value: V) {
        send(CacheCommand.Put(key, value))
    }
    
    fun remove(key: K) {
        send(CacheCommand.Remove(key))
    }
    
    fun clear() {
        send(CacheCommand.Clear)
    }
}
```

### Pattern 2: Flow Operators для сложных сценариев

Создайте `shared/src/commonMain/kotlin/com.skillsync/concurrency/FlowExtensions.kt`:

```kotlin
package com.skillsync.concurrency

import kotlinx.coroutines.flow.*

// Debounce для поиска (игнорирует быстрые события)
fun <T> Flow<T>.debounce(delayMillis: Long): Flow<T> {
    return this.debounce(delayMillis)
}

// Retry с exponential backoff
suspend fun <T> retry(
    times: Int = 3,
    delay: suspend (attempt: Int) -> Long = { attempt -> 
        (500L * 2.pow(attempt - 1)) // 500ms, 1s, 2s
    },
    block: suspend () -> T
): T {
    repeat(times) { attempt ->
        try {
            return block()
        } catch (e: Exception) {
            if (attempt == times - 1) throw e
            delay(delay(attempt))
        }
    }
    throw RuntimeException("Max retries exceeded")
}

// Circuit Breaker pattern
class CircuitBreaker(
    private val failureThreshold: Int = 5,
    private val resetTimeout: Long = 60_000
) {
    private var failures = 0
    private var lastFailureTime: Long = 0
    private var isOpen = false
    
    @Synchronized
    suspend fun <T> execute(block: suspend () -> T): T {
        if (isOpen) {
            // Проверяем можно ли сбросить circuit breaker
            if (System.currentTimeMillis() - lastFailureTime > resetTimeout) {
                isOpen = false
                failures = 0
            } else {
                throw CircuitBreakerOpenException()
            }
        }
        
        try {
            val result = block()
            failures = 0 // Сбрасываем счетчик при успехе
            return result
        } catch (e: Exception) {
            failures++
            lastFailureTime = System.currentTimeMillis()
            
            if (failures >= failureThreshold) {
                isOpen = true
            }
            
            throw e
        }
    }
}

class CircuitBreakerOpenException : Exception("Circuit breaker is open")

// Cache Flow с TTL
fun <T> Flow<T>.cacheWithTtl(
    ttl: Long = 5_000,
    scope: CoroutineScope = CoroutineScope(SupervisorJob())
): StateFlow<T> {
    return channel
        .stateIn(
            scope = scope,
            started = SharingStarted.WhileSubscribed(ttl),
            initialValue = throw IllegalStateException("No initial value")
        )
}

// Merge multiple flows с приоритетом
fun <T> mergeWithPriority(vararg flows: Flow<T>): Flow<T> {
    return flows.merge()
}

// Rate limiting для API вызовов
class RateLimiter(
    private val rate: Int, // запросов в секунду
    private val windowSize: Long = 1_000
) {
    private val timestamps = mutableListOf<Long>()
    
    @Synchronized
    suspend fun acquire() {
        val now = System.currentTimeMillis()
        
        // Удаляем старые timestamps вне окна
        timestamps.removeAll { now - it > windowSize }
        
        if (timestamps.size >= rate) {
            // Ждем пока освободится место в окне
            val waitTime = timestamps[0] + windowSize - now
            if (waitTime > 0) {
                delay(waitTime)
            }
            
            // Снова очищаем после ожидания
            val newNow = System.currentTimeMillis()
            timestamps.removeAll { newNow - it > windowSize }
        }
        
        timestamps.add(System.currentTimeMillis())
    }
}

// Bulkhead pattern - изоляция ресурсов
class Bulkhead<T>(
    private val maxConcurrent: Int = 10,
    private val scope: CoroutineScope = CoroutineScope(SupervisorJob())
) {
    private var activeRequests = 0
    
    @Synchronized
    suspend fun <R> execute(block: suspend () -> R): R {
        while (activeRequests >= maxConcurrent) {
            delay(100) // Ждем пока освободится слот
        }
        
        activeRequests++
        
        return try {
            block()
        } finally {
            activeRequests--
        }
    }
}
```

---

## 4. Advanced Build Configuration

### Оптимизация сборки для production

Создайте `shared/build.gradle.kts` (production configuration):

```kotlin
plugins {
    kotlin("multiplatform")
    id("com.android.library")
}

kotlin {
    // ... existing configuration
    
    // Оптимизация для production
    targets.all {
        compilations.all {
            kotlinOptions {
                // Включаем оптимизации
                freeCompilerArgs += listOf(
                    "-Xopt-in=kotlin.RequiresOptIn",
                    "-Xjvm-default=all" // Для JVM target
                )
            }
        }
    }
    
    // Native optimization для iOS/Android Native
    applyDefaultHierarchyTemplate {
        common {
            group("native") {
                withAndroidTarget()
                withIosTargets()
            }
        }
    }
}

// Android optimization
android {
    buildTypes {
        getByName("release") {
            isMinifyEnabled = true
            isShrinkResources = true
            
            // ProGuard/R8 rules
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    
    // NDK optimization
    defaultConfig {
        ndk {
            abiFilters += listOf("armeabi-v7a", "arm64-v8a", "x86", "x86_64")
        }
    }
}

// Gradle performance optimization
tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompile> {
    kotlinOptions {
        // Incremental compilation
        freeCompilerArgs += "-Xincremental"
    }
}

// Build cache для ускорения
buildscript {
    dependencies {
        classpath("org.gradle:gradle-build-cache-http:8.0")
    }
}

// Remote build cache (для CI/CD)
buildCache {
    remote<HttpBuildCache> {
        url = uri("https://gradle-cache.yourcompany.com/")
        isPush = findProperty("pushCache") == "true"
    }
}
```

Создайте `gradle.properties`:

```properties
# Gradle performance
org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=512m
org.gradle.caching=true
org.gradle.configureondemand=true
org.gradle.parallel=true

# Kotlin/Native optimization
kotlin.native.cacheKind=none # Отключаем cache для CI

# Android
android.useAndroidX=true
android.enableJetifier=false

# KMP specific
kotlin.mpp.stability.nowarn=true
```

---

## 📝 Домашнее задание (Модуль 8)

### Задача: Создание production-ready KMP приложения с advanced features

**Требования:**

1. **C++ Interop:**
   - Интегрируйте OpenCV или другую C++ библиотеку для обработки изображений
   - Реализуйте expect/actual для Android и iOS
   - Напишите unit-тесты для Kotlin обертки

2. **Machine Learning:**
   - Интегрируйте TensorFlow Lite или Core ML для классификации изображений
   - Реализуйте fallback strategy (ML на устройстве → серверный API)
   - Добавьте UI для демонстрации ML возможностей

3. **Advanced Concurrency:**
   - Реализуйте Actor pattern для управления shared state
   - Добавьте Circuit Breaker для network вызовов
   - Используйте Rate Limiter для API rate limiting

4. **Build Optimization:**
   - Настройте production build с minification и obfuscation
   - Оптимизируйте время сборки (incremental builds, caching)
   - Добавьте build metrics и monitoring

5. **Production Readiness:**
   - Настройте crash reporting (Sentry, Firebase Crashlytics)
   - Добавьте analytics (Firebase Analytics, Mixpanel)
   - Реализуйте feature flags для A/B тестирования

**Критерии сдачи:**
- ✅ C++ библиотека интегрирована и работает на обеих платформах
- ✅ ML модель загружается и делает предсказания
- ✅ Actor pattern используется для управления состоянием
- ✅ Circuit Breaker защищает от cascading failures
- ✅ Production build оптимизирован (размер APK < 30MB)
- ✅ Crash reporting и analytics настроены

**Бонусные задания:**
- Добавьте support для watchOS/tvOS
- Реализуйте offline-first архитектуру с background sync
- Напишите техническую документацию архитектуры (ARCHITECTURE.md)

---

## 💡 Советы по выполнению

1. **Начните с малого:** Сначала интегрируйте одну C++ функцию, потом масштабируйте.
2. **Тестируйте на обеих платформах:** C++ interop может вести себя по-разному на Android/iOS.
3. **Используйте profiling:** Instruments (iOS), Profiler (Android) для поиска bottlenecks.
4. **Документируйте всё:** C++ interop требует четкой документации API.
5. **Не усложняйте prematurely:** Используйте server-side ML если on-device слишком сложно.

---

## 🎓 Завершение Senior Track

**Поздравляю! Вы завершили Senior Track по KMP!**

### Что вы освоили:
✅ **Module 1:** Основы KMP, настройка окружения  
✅ **Module 2:** UI с Compose Multiplatform и SwiftUI  
✅ **Module 3:** Database (SQLDelight, Realm)  
✅ **Module 4:** Networking (Ktor Client, expect/actual)  
✅ **Module 5:** Dependency Injection (Koin, inject)  
✅ **Module 6:** Testing & Quality (Unit, Integration, UI tests, CI/CD)  
✅ **Module 7:** Architecture Patterns (Clean Arch, MVI, Modularization)  
✅ **Module 8:** Advanced Topics (C++ Interop, ML, Concurrency)

### Следующие шаги:
1. **Портфолио:** Опубликуйте SkillSync на GitHub с README и demo
2. **Production:** Запустите реальное KMP приложение в production
3. **Community:** Вносите вклад в open-source KMP проекты
4. **Mentorship:** Помогайте другим разработчикам в изучении KMP

### Ресурсы для дальнейшего обучения:
- 📚 [Kotlin Multiplatform Documentation](https://www.jetbrains.com/help/kotlin-multiplatform-dev/)
- 📚 [KMP Samples от JetBrains](https://github.com/JetBrains/compose-multiplatform-samples)
- 📚 [KMP Slack Community](https://kotlinlang.slack.com/)
- 📚 [KotlinConf Talks о KMP](https://kotlinconf.com/)

**Удачи в вашей карьере Senior Kotlin Multiplatform разработчика! 🚀**

# Модуль 5: AI в нативных интеграциях

## 📋 Обзор модуля

**Продолжительность:** 12 часов  
**Сложность:** Advanced  
**Цель:** Использование AI для генерации нативного кода (Swift/Kotlin), интеграции платформенных API и ML-библиотек

---

## 🎯 Цели обучения

После прохождения модуля вы сможете:
- ✅ Генерировать **нативный код** (Swift для iOS, Kotlin для Android) с AI
- ✅ Интегрировать **платформенные API** (Camera, Location, Notifications)
- ✅ Использовать **ML Kit и Core ML** с AI-ассистентом
- ✅ Создавать **платформенные extensions** и native modules

---

## 📚 Темы модуля

### Тема 5.1: Генерация нативного кода для Android (2 часа)

#### Шаблон prompt'а для Android нативного кода:

```
Ты - Senior Android разработчик с 7+ лет опытом.

Задача:
Создай нативную Android реализацию для [ФУНКЦИОНАЛ].

Контекст:
- KMP приложение с expect/actual pattern
- Target API: Android [VERSION]+ (minSdk 24)
- Используем Kotlin для actual implementation

Требования:
1. Создай actual class который implements expect interface из commonMain
2. Используй современные Android APIs (Jetpack libraries)
3. Обработай permissions и lifecycle
4. Включи error handling и logging

[ОПИСАНИЕ ФУНКЦИОНАЛА]

Создай:
1. Actual implementation class
2. Необходимые Android permissions в AndroidManifest.xml
3. Permission request flow (если нужно)
4. Пример использования из common code

Следуй Android best practices:
- Lifecycle-aware components
- Coroutines для async operations  
- Result sealed class для error handling

Покажи полный код.
```

---

#### Пример: Генерация Camera API для Android

**Промпт:**
```
Создай actual implementation для CameraService на Android.

Expect interface из commonMain:
```kotlin
interface CameraService {
    suspend fun takePhoto(): Result<ByteArray>
    suspend fun startVideoRecording(): Result<Unit>
    suspend fun stopVideoRecording(): Result<String> // returns file path
    fun getCameraPreview(): Flow<Bitmap>
}
```

Требования:
- Используй CameraX API (latest stable)
- Обработай CAMERA permission
- Поддерживай front/back camera selection
- Валидируй hardware capabilities

Создай полный actual implementation.
```

**Ожидаемый результат от AI:**

```kotlin
// androidMain/platform/CameraService.android.kt
package platform

import android.Manifest
import android.content.Context
import android.content.pm.PackageManager
import android.graphics.Bitmap
import androidx.camera.core.*
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.core.content.ContextCompat
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow

actual class CameraService actual constructor(
    private val context: Context,
    private val cameraSelector: CameraSelector = CameraSelector.DEFAULT_BACK_CAMERA
) : CameraService {
    
    private var cameraProvider: ProcessCameraProvider? = null
    private var camera: Camera? = null
    private val previewFlow = MutableStateFlow<Bitmap?>(null)
    
    init {
        initializeCameraProvider()
    }
    
    private fun initializeCameraProvider() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(context)
        // ... инициализация CameraX
    }
    
    actual suspend fun takePhoto(): Result<ByteArray> {
        return try {
            if (!hasCameraPermission()) {
                return Result.Error(CameraError.PermissionDenied)
            }
            
            // CameraX photo capture implementation
            val photo = capturePhoto()
            Result.Success(photo)
        } catch (e: Exception) {
            Result.Error(CameraError.Unknown(e.message))
        }
    }
    
    actual suspend fun startVideoRecording(): Result<Unit> {
        // Video recording implementation с CameraX
    }
    
    actual suspend fun stopVideoRecording(): Result<String> {
        // Stop recording и вернуть путь к файлу
    }
    
    actual fun getCameraPreview(): Flow<Bitmap> {
        return previewFlow.asStateFlow()
    }
    
    private fun hasCameraPermission(): Boolean {
        return ContextCompat.checkSelfPermission(
            context, 
            Manifest.permission.CAMERA
        ) == PackageManager.PERMISSION_GRANTED
    }
}

actual fun CameraService(
    context: Context,
    cameraSelector: CameraSelector
): CameraService {
    return CameraService(context, cameraSelector)
}

sealed class CameraError : Exception() {
    object PermissionDenied : CameraError("Camera permission not granted")
    data class Unknown(val message: String) : CameraError(message)
}
```

---

### Тема 5.2: Генерация нативного кода для iOS (3 часа)

#### Шаблон prompt'а для iOS нативного кода:

```
Ты - Senior iOS разработчик с 7+ лет опытом в Swift.

Задача:
Создай нативную iOS реализацию для [ФУНКЦИОНАЛ] используя Kotlin/Native.

Контекст:
- KMP приложение с expect/actual pattern  
- iOS 15.0+ (min deployment target)
- Используем Kotlin Multiplatform для actual implementation

Требования:
1. Создай actual class который implements expect interface из commonMain
2. Интегрируй соответствующие iOS frameworks (AVFoundation, CoreLocation, etc.)
3. Обработай permissions через Info.plist и runtime requests
4. Обработай delegate patterns и completion handlers

[ОПИСАНИЕ ФУНКЦИОНАЛА]

Создай:
1. Actual implementation в Kotlin (iosMain)
2. Необходимые ключи в Info.plist
3. Permission request flow
4. Bridge code для взаимодействия с Swift (если нужно)

Используй:
- Kotlin coroutines для async operations
- Result sealed class для error handling  
- Proper disposal of native resources

Покажи полный код.
```

---

#### Пример: Генерация Core Location для iOS

**Промпт:**
```
Создай actual implementation для LocationService на iOS.

Expect interface:
```kotlin
interface LocationService {
    suspend fun getCurrentLocation(): Result<Location>
    fun getLocationUpdates(): Flow<Location>
    suspend fun requestPermission(): PermissionStatus
}

data class Location(
    val latitude: Double,
    val longitude: Double,  
    val accuracy: Double,
    val timestamp: Long
)
```

Требования:
- Используй CoreLocation framework (CLLocationManager)
- Обработай LOCATION_ALWAYS и LOCATION_WHILE_USING permissions
- Поддерживай background location updates (если configured)
- Обработай location services disabled scenario

Создай полный actual implementation в Kotlin.
```

**Ожидаемый результат:** AI сгенерирует полный код с CLLocationManager, delegate pattern, permission handling.

---

### Тема 5.3: Интеграция ML Kit и Core ML (3 часа)

#### Шаблон prompt'а для ML интеграции:

```
Ты - Mobile ML Engineer с опытом в on-device machine learning.

Задача:
Интегрируй [ML ФУНКЦИОНАЛ] в KMP приложение.

Контекст:
- Android: ML Kit (Google)
- iOS: Core ML + Create ML

Требования к expect interface (commonMain):
1. Методы для inference: predict(input): Result<Output>
2. Model loading и initialization  
3. Error handling для missing models, invalid input

Требования к actual implementations:
- Android: Используй ML Kit [MODEL TYPE] API
- iOS: Используй Core ML .mlmodelc

[ОПИСАНИЕ ML ФУНКЦИОНАЛА - например: Image Classification, Text Recognition, Object Detection]

Создай:
1. Expect interface в commonMain с input/output data classes
2. Android actual implementation с ML Kit
3. iOS actual implementation с Core ML  
4. Instructions для добавления model файлов в проекты

Покажи полный код и configuration files.
```

---

#### Пример: Text Recognition (OCR) с ML Kit / Vision Framework

**Промпт:**
```
Создай expect/actual для OCR (text recognition из изображений).

Android: ML Kit Text Recognition  
iOS: Vision Framework VNRecognizeTextRequest

Expect interface должен возвращать:
- Recognized text (String)
- Text blocks с bounding boxes  
- Confidence scores

Обработай разные языки (multi-language support).
```

---

### Тема 5.4: Создание платформенных extensions (2 часа)

#### Шаблон prompt'а для native modules:

```
Ты - Senior Mobile Engineer, эксперт в нативных модулях.

Задача:
Создай нативный module для [ФУНКЦИОНАЛ] который будет доступен из KMP common code.

Контекст:
- Функционал недоступен в стандартных KMP библиотеках
- Нужно использовать платформенно-специфичные APIs

Требования:
1. Android: Создай Kotlin class с proper lifecycle management
2. iOS: Создай Swift class + Kotlin bridge (или чистый Kotlin/Native)
3. Expect/actual interface в commonMain для unified API

[ОПИСАНИЕ ФУНКЦИОНАЛА - например: Deep Linking, In-App Purchases, Push Notifications]

Создай полный implementation для обеих платформ.
```

---

## 📝 Практические задания модуля

### Задание 5.1: QR Code Scanner с AI (3 часа)

**Требования:**
- Генерируй expect/actual для QR scanning
- Android: ML Kit Barcode Scanning
- iOS: AVFoundation + Vision

**Критерии:**
- ✓ Работает на обеих платформах
- ✓ Обработка permissions

---

### Задание 5.2: Push Notifications с AI (3 часа)

**Требования:**
- Интеграция FCM для Android
- Интеграция APNs для iOS
- Unified API в commonMain

**Критерии:**
- ✓ Получение device tokens
- ✓ Обработка push сообщений

---

### Задание 5.3: Image Processing с ML (3 часа)

**Требования:**
- Face detection или image classification
- ML Kit для Android, Core ML для iOS

**Критерии:**
- ✓ Model loading и inference
- ✓ Error handling

---

### Задание 5.4: Платформенный extension (3 часа)

**Требования:**
- Выберите нативную фичу (Biometric Auth, Haptic Feedback, etc.)
- Создайте unified KMP API

**Критерии:**
- ✓ Clean expect/actual abstraction
- ✓ Документация API

---

## 🚫 Ошибки при нативной интеграции с AI

### ❌ AI использует устаревшие iOS/Android APIs
**Решение:** Указывайте: "Используй latest stable APIs для iOS 17+ / Android 14+"

### ❌ AI не обрабатывает permissions properly
**Решение:** Явно просите: "Включи полный permission request flow с fallback"

### ❌ AI создает memory leaks в native code
**Решение:** Просите: "Обеспечь proper cleanup и disposal of native resources"

---

## 📚 Дополнительные материалы

### Документация:
- [ML Kit Documentation](https://developers.google.com/ml-kit)
- [Core ML Documentation](https://developer.apple.com/coreml/)
- [CameraX Documentation](https://developer.android.com/training/camerax)

### Инструменты:
- [Create ML](https://developer.apple.com/create-ml/) - Создание Core ML моделей
- [Netron](https://netron.app/) - Визуализация ML моделей

### Примеры:
- [KMP Samples - Native Features](https://github.com/JetBrains/compose-multiplatform-samples)

---

## 🚀 Следующий шаг

Переходите к [Модулю 6](../Module_06_AI_Production/content.md): AI в production

**Время до следующего модуля:** 2-3 недели  
**Рекомендуемая практика:** Интегрируйте минимум 1 нативную фичу с AI

---

**Удачи в нативной интеграции с помощью ИИ! 📱🤖**

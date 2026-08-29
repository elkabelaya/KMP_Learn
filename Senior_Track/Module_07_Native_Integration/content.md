# 📘 Модуль 4: Нативные интеграции

**Добро пожаловать в четвертый модуль Senior Track!**  
В этом модуле мы выйдем за пределы Kotlin и научимся интегрировать нативные библиотеки C, C++, Objective-C и Swift. Вы поймете, как работать с Kotlin/Native interop и создадите мосты между KMP кодом и платформенными API.

**Цели модуля:**
1. Освоить C-interop в Kotlin/Native
2. Научиться создавать Objective-C/Swift биндинги для iOS
3. Интегрировать нативные библиотеки (MapKit, ARKit, CameraX)
4. Создать кроссплатформенные абстракции для нативных API через Expect/Actual
5. Реализовать AR-фильтры и работу с камерой

**Время выполнения:** ~30–40 часов (6 недель).

---

## 1. Введение: Когда нужны нативные интеграции?

KMP покрывает 80% потребностей, но что делать с остальными 20%?

### Сценарии использования нативных интеграций:

#### 🗺️ Геолокация и карты
- iOS: MapKit (нативный фреймворк)
- Android: Google Maps SDK

#### 📷 Камера и обработка изображений
- iOS: AVFoundation, CoreImage
- Android: CameraX

#### 🎮 AR/VR
- iOS: ARKit
- Android: ARCore

#### 🔐 Биометрия и безопасность
- iOS: LocalAuthentication, Secure Enclave
- Android: BiometricPrompt, StrongBox Keystore

#### 📊 Сторонние SDK
- Analytics (Firebase, Amplitude)
- Ads (AdMob, iAd)
- Payment (Stripe, PayPal)

---

## 2. Теория: C-interop в Kotlin/Native

### Что такое C-interop?
Механизм, позволяющий вызывать функции из C-библиотек напрямую из Kotlin/Native кода.

### Как это работает?

```
Kotlin/Native код → C-interop layer → C-библиотека (.so/.dylib/.a)
```

### Шаг 1: Подготовка C-библиотеки

Допустим, у нас есть библиотека для обработки изображений `libvips`:

```c
// vips.h
#ifndef VIPS_H
#define VIPS_H

typedef struct {
    int width;
    int height;
    unsigned char* data;
} VipsImage;

// Функция для сжатия изображения
VipsImage* vips_compress(VipsImage* image, int quality);

// Функция для поворота
VipsImage* vips_rotate(VipsImage* image, int degrees);

// Освобождение памяти
void vips_image_free(VipsImage* image);

#endif
```

### Шаг 2: Настройка C-interop в build.gradle.kts

Создайте `shared/src/iosMain/build.gradle.kts`:

```kotlin
plugins {
    kotlin("native.cocoapods")
}

kotlin {
    sourceSets {
        val iosMain by getting {
            kotlinOptions {
                // Путь к C-хедерам
                freeCompilerArgs += listOf(
                    "-Xx-source-gen",
                    "-P", "kotlin.native.cinterop.defines=VIPS_VERSION=8.10"
                )
            }
        }
    }
}

// Настройка CocoaPods для iOS
cocoapods {
    pods {
        pod('VIPSDependency', '>= 8.10')
    }
}
```

### Шаг 3: Генерация биндингов

Создайте `shared/src/iosMain/kotlin/cinterop/vips/README.md` с инструкцией:

```bash
# Генерация биндингов из C-хедеров
kotlin-native-utils generate-bindings \
  --headers=/path/to/vips.h \
  --output=shared/src/iosMain/kotlin/cinterop/vips

# Или через Gradle
./gradlew :shared:iosX64GenerateNativeCodeVips
```

### Шаг 4: Использование в Kotlin коде

Создайте `shared/src/iosMain/kotlin/platform/ImageProcessor.kt`:

```kotlin
package platform

import cinterop.vips.*

class ImageProcessor {
    fun compress(image: ByteArray, quality: Int): ByteArray {
        // Конвертируем Kotlin ByteArray в C-структуру
        val vipsImage = VipsImage(
            width = image.width,
            height = image.height,
            data = image.toCPointer()
        )
        
        // Вызываем C-функцию
        val compressed = vips_compress(vipsImage, quality)
        
        // Конвертируем обратно в Kotlin ByteArray
        val result = compressed.data.toKotlinByteArray()
        
        // Освобождаем память (важно для C!)
        vips_image_free(compressed)
        
        return result
    }
    
    fun rotate(image: ByteArray, degrees: Int): ByteArray {
        val vipsImage = VipsImage(
            width = image.width,
            height = image.height,
            data = image.toCPointer()
        )
        
        val rotated = vips_rotate(vipsImage, degrees)
        val result = rotated.data.toKotlinByteArray()
        
        vips_image_free(rotated)
        
        return result
    }
}

// Extension функции для конвертации
private fun ByteArray.toCPointer(): CPointer<ByteVar> {
    // Реализация конвертации (упрощенно)
    return this.toCValues()
}

private fun CPointer<ByteVar>.toKotlinByteArray(): ByteArray {
    // Реализация конвертации (упрощенно)
    return this.toKotlinArray()
}
```

---

## 3. Теория: Objective-C/Swift биндинги для iOS

### Автоматическая генерация через Kotlin/Native

Kotlin/Native автоматически генерирует биндинги для iOS фреймворков.

### Доступ к нативным API:

```kotlin
// iOS - доступ к MapKit
import platform.MapKit.MKMapView
import platform.MapKit.MKCoordinateRegion
import platform.CoreLocation.CLCoordinates

class MapViewController {
    private val mapView = MKMapView()
    
    fun setRegion(latitude: Double, longitude: Double, zoom: Double) {
        val coordinate = CLLocationCoordinate2D(latitude, longitude)
        val region = MKCoordinateRegion(
            center = coordinate,
            latitudinalMeters = zoom,
            longitudinalMeters = zoom
        )
        
        mapView.setRegion(region, animated = true)
    }
}
```

### Создание кастомных биндингов через @ObjCName

```kotlin
@ObjCName("SkillSyncImageProcessor", exact = true)
class ImageProcessor @Constructor:{}() {
    @ObjCName("compressImage:quality:")
    fun compress(image: UIImage, quality: Int): UIImage {
        // Реализация на Swift/Kotlin
    }
}
```

---

## 4. Практика: Создание кроссплатформенной абстракции для карт

### Шаг 1: Expect/Actual контракт в commonMain

Создайте `shared/src/commonMain/kotlin/platform/MapProvider.kt`:

```kotlin
package platform

interface MapProvider {
    fun showLocation(latitude: Double, longitude: Double)
    fun addMarker(id: String, latitude: Double, longitude: Double, title: String)
    fun removeMarker(id: String)
    fun setCamera(latitude: Double, longitude: Double, zoom: Float)
}

expect fun createMapProvider(): MapProvider

interface MapMarker {
    val id: String
    val latitude: Double
    val longitude: Double
    val title: String
}

expect fun createMapMarker(id: String, latitude: Double, longitude: Double, title: String): MapMarker
```

### Шаг 2: Реализация для iOS (MapKit)

Создайте `shared/src/iosMain/kotlin/platform/MapProvider.kt`:

```kotlin
package platform

import androidx.compose.ui.window.ComposeUIViewController
import platform.MapKit.*
import platform.UIKit.UIColor

actual fun createMapProvider(): MapProvider = IosMapProvider()

class IosMapProvider : MapProvider {
    private val mapView = MKMapView(frame = CGRect(0.0, 0.0, 400.0, 300.0))
    private val markers = mutableMapOf<String, MKAnnotation>()
    
    init {
        mapView.mapType = MKMapTypeStandard
    }
    
    actual fun showLocation(latitude: Double, longitude: Double) {
        val coordinate = CLLocationCoordinate2D(latitude, longitude)
        val region = MKCoordinateRegion(
            center = coordinate,
            latitudinalMeters = 1000.0,
            longitudinalMeters = 1000.0
        )
        
        mapView.setRegion(region, animated = true)
    }
    
    actual fun addMarker(id: String, latitude: Double, longitude: Double, title: String) {
        val annotation = MKPointAnnotation()
        annotation.coordinate = CLLocationCoordinate2D(latitude, longitude)
        annotation.title = title
        
        mapView.addAnnotation(annotation)
        markers[id] = annotation
    }
    
    actual fun removeMarker(id: String) {
        markers.remove(id)?.let { mapView.removeAnnotation(it) }
    }
    
    actual fun setCamera(latitude: Double, longitude: Double, zoom: Float) {
        val coordinate = CLLocationCoordinate2D(latitude, longitude)
        val region = MKCoordinateRegion(
            center = coordinate,
            latitudinalMeters = (1000 / zoom).toDouble(),
            longitudinalMeters = (1000 / zoom).toDouble()
        )
        
        mapView.setRegion(region, animated = true)
    }
}

actual fun createMapMarker(id: String, latitude: Double, longitude: Double, title: String): MapMarker {
    return object : MapMarker {
        override val id = id
        override val latitude = latitude
        override val longitude = longitude
        override val title = title
    }
}
```

### Шаг 3: Реализация для Android (Google Maps)

Создайте `shared/src/androidMain/kotlin/platform/MapProvider.kt`:

```kotlin
package platform

import com.google.android.gms.maps.GoogleMap
import com.google.android.gms.maps.model.LatLng
import com.google.android.gms.maps.model.MarkerOptions

actual fun createMapProvider(): MapProvider = AndroidMapProvider()

class AndroidMapProvider(private val googleMap: GoogleMap) : MapProvider {
    private val markers = mutableMapOf<String, com.google.android.gms.maps.model.Marker>()
    
    actual fun showLocation(latitude: Double, longitude: Double) {
        googleMap.moveCamera(
            com.google.android.gms.maps.CameraUpdateFactory.newLatLngZoom(
                LatLng(latitude, longitude),
                15f
            )
        )
    }
    
    actual fun addMarker(id: String, latitude: Double, longitude: Double, title: String) {
        val marker = googleMap.addMarker(
            MarkerOptions()
                .position(LatLng(latitude, longitude))
                .title(title)
        )
        
        marker?.let { markers[id] = it }
    }
    
    actual fun removeMarker(id: String) {
        markers.remove(id)?.remove()
    }
    
    actual fun setCamera(latitude: Double, longitude: Double, zoom: Float) {
        googleMap.moveCamera(
            com.google.android.gms.maps.CameraUpdateFactory.newLatLngZoom(
                LatLng(latitude, longitude),
                zoom
            )
        )
    }
}

actual fun createMapMarker(id: String, latitude: Double, longitude: Double, title: String): MapMarker {
    return object : MapMarker {
        override val id = id
        override val latitude = latitude
        override val longitude = longitude
        override val title = title
    }
}
```

### Шаг 4: Использование в commonMain коде

Создайте `shared/src/commonMain/kotlin/ui/screens/MapScreen.kt`:

```kotlin
package ui.screens

import androidx.compose.runtime.*
import platform.MapProvider
import platform.createMapProvider

@Composable
fun MapScreen(
    initialLat: Double,
    initialLng: Double
) {
    val mapProvider = remember { createMapProvider() }
    
    // Добавляем маркеры (общий код!)
    LaunchedEffect(Unit) {
        mapProvider.addMarker("1", 55.7558, 37.6173, "Москва")
        mapProvider.addMarker("2", 59.9311, 30.3609, "Санкт-Петербург")
        mapProvider.addMarker("3", 50.4501, 30.5239, "Киев")
        
        mapProvider.showLocation(55.7558, 37.6173)
    }
    
    // Нативный виджет карты (через Compose Interop)
    NativeMapView(provider = mapProvider)
}

@Composable
expect fun NativeMapView(
    provider: MapProvider,
    modifier: Modifier = Modifier
)
```

---

## 5. Практика: Интеграция AR-фильтров (ARKit/ARCore)

### Шаг 1: Expect/Actual для AR

Создайте `shared/src/commonMain/kotlin/platform/ArProvider.kt`:

```kotlin
package platform

sealed class ArTrackingState {
    object Initializing : ArTrackingState()
    object Tracking : ArTrackingState()
    object Lost : ArTrackingState()
}

interface ArProvider {
    fun startSession()
    fun stopSession()
    fun placeVirtualObject(x: Float, y: Float, z: Float)
    
    val trackingState: ArTrackingState
}

expect fun createArProvider(): ArProvider? // Nullable - не все платформы поддерживают AR
```

### Шаг 2: Реализация для iOS (ARKit)

Создайте `shared/src/iosMain/kotlin/platform/ArProvider.kt`:

```kotlin
package platform

import platform.ARKit.ARSession
import platform.ARKit.ARWorldTrackingConfiguration
import platform.ARKit.ARAnchor

actual fun createArProvider(): ArProvider? {
    // Проверяем поддержку ARKit
    if (!ARWorldTrackingConfiguration.isSupported) {
        return null
    }
    
    return IosArProvider()
}

class IosArProvider : ArProvider {
    private val session = ARSession.sharedInstance()
    private val configuration = ARWorldTrackingConfiguration()
    
    actual var trackingState: ArTrackingState = ArTrackingState.Initializing
    
    actual fun startSession() {
        session.run(configuration, restoreState = false)
        
        // Наблюдаем за состоянием сессии
        session.delegate = object : ARSessionDelegateProtocol {
            override fun session(
                session: ARSession, 
                didFailWithError: platform.Foundation.NSError?
            ) {
                trackingState = ArTrackingState.Lost
            }
            
            override fun session(
                session: ARSession, 
                didAddAnchors: List<ARAnchor>
            ) {
                trackingState = ArTrackingState.Tracking
            }
        }
    }
    
    actual fun stopSession() {
        session.pause()
    }
    
    actual fun placeVirtualObject(x: Float, y: Float, z: Float) {
        // Размещаем виртуальный объект в AR-пространстве
        // Реализация зависит от конкретного AR фреймворка
    }
}

actual fun createArProvider(): ArProvider? = createArProvider()
```

### Шаг 3: Реализация для Android (ARCore)

Создайте `shared/src/androidMain/kotlin/platform/ArProvider.kt`:

```kotlin
package platform

import com.google.ar.core.ArCoreApk
import com.google.ar.core.Config
import com.google.ar.core.Session

actual fun createArProvider(): ArProvider? {
    // Проверяем поддержку ARCore
    if (ArCoreApk.getInstance().checkAvailability(context) != ArCoreApk.Availability.SUCCESS) {
        return null
    }
    
    return AndroidArProvider()
}

class AndroidArProvider : ArProvider {
    private var session: Session? = null
    
    actual var trackingState: ArTrackingState = ArTrackingState.Initializing
    
    actual fun startSession() {
        try {
            session = Session(context, ArCoreApk.getInstance())
            
            val config = Config(session!!)
            config.focusMode = Config.FocusMode.AUTO
            session!!.configure(config)
            
            trackingState = ArTrackingState.Tracking
        } catch (e: Exception) {
            trackingState = ArTrackingState.Lost
        }
    }
    
    actual fun stopSession() {
        session?.pause()
    }
    
    actual fun placeVirtualObject(x: Float, y: Float, z: Float) {
        // Размещаем виртуальный объект в AR-пространстве
    }
}

actual fun createArProvider(): ArProvider? = createArProvider()
```

---

## 6. Практика: Работа с камерой (CameraX/AVFoundation)

### Шаг 1: Expect/Actual для камеры

Создайте `shared/src/commonMain/kotlin/platform/CameraProvider.kt`:

```kotlin
package platform

sealed class CameraFrame {
    data class Image(val bytes: ByteArray, val width: Int, val height: Int) : CameraFrame()
}

interface CameraProvider {
    fun startPreview(onFrameReceived: (CameraFrame) -> Unit)
    fun stopPreview()
    fun takePhoto(): Result<ByteArray>
}

expect fun createCameraProvider(): CameraProvider?
```

### Шаг 2: Реализация для Android (CameraX)

Создайте `shared/src/androidMain/kotlin/platform/CameraProvider.kt`:

```kotlin
package platform

import androidx.camera.core.*
import androidx.camera.lifecycle.ProcessCameraProvider

actual fun createCameraProvider(): CameraProvider? = AndroidCameraProvider()

class AndroidCameraProvider : CameraProvider {
    private var cameraProvider: ProcessCameraProvider? = null
    private var imageCapture: ImageCapture? = null
    
    actual fun startPreview(onFrameReceived: (CameraFrame) -> Unit) {
        // Настройка CameraX preview с анализом кадров
        val imageAnalysis = ImageAnalysis.Builder()
            .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
            .build()
            .also {
                it.setAnalyzer(ContextContext.getLifecycleOwner()) { imageProxy ->
                    val bytes = imageProxy.planes[0].buffer.toByteArray()
                    onFrameReceived(CameraFrame.Image(
                        bytes = bytes,
                        width = imageProxy.width,
                        height = imageProxy.height
                    ))
                    imageProxy.close()
                }
            }
        
        // Биндинг камерных use cases
        cameraProvider?.bindToLifecycle(
            /* lifecycleOwner */,
            CameraSelector.DEFAULT_BACK_CAMERA,
            Preview.Builder().build(),
            imageAnalysis
        )
    }
    
    actual fun stopPreview() {
        cameraProvider?.unbindAll()
    }
    
    actual fun takePhoto(): Result<ByteArray> {
        return try {
            val photoFile = File(context.cacheDir, "photo_${System.currentTimeMillis()}.jpg")
            
            val outputOptions = ImageCapture.OutputFileOptions(photoFile)
            imageCapture?.takePicture(outputOptions, ContextContext.getLifecycleOwner()) { 
                // Обработка результата
            }
            
            Result.Success(photoFile.readBytes())
        } catch (e: Exception) {
            Result.Failure(e)
        }
    }
}
```

### Шаг 3: Реализация для iOS (AVFoundation)

Создайте `shared/src/iosMain/kotlin/platform/CameraProvider.kt`:

```kotlin
package platform

import platform.AVFoundation.*

actual fun createCameraProvider(): CameraProvider? = IosCameraProvider()

class IosCameraProvider : CameraProvider {
    private val captureSession = AVCaptureSession()
    
    actual fun startPreview(onFrameReceived: (CameraFrame) -> Unit) {
        // Настройка AVCaptureSession для iOS
        val videoDevice = AVCaptureDevice.default(
            AVCaptureDevice.DeviceType.BuiltInWideAngleCamera,
            AVCaptureDevice.Position.Back,
            nil
        )
        
        // Настройка видео потока и обработчика кадров
        val videoDataOutput = AVCaptureVideoDataOutput()
        videoDataOutput.alwaysDiscardsLateVideoFrames = true
        
        val queue = dispatch_queue_create("cameraQueue", null)
        
        videoDataOutput.setSampleBufferDelegate(
            object : AVCaptureVideoDataOutputSampleBufferDelegateProtocol {
                override fun captureOutput(
                    output: AVCaptureOutput, 
                    didOutputSampleBuffer: CMSampleBuffer?, 
                    connection: AVCaptureConnection?, 
                    connectionToDevice: AVCaptureInputPort?, 
                    session: AVCaptureSession?
                ) {
                    // Обработка кадра и вызов callback
                    sampleBufferToByteArray(didOutputSampleBuffer)?.let { bytes ->
                        onFrameReceived(CameraFrame.Image(
                            bytes = bytes,
                            width = 1920,
                            height = 1080
                        ))
                    }
                }
            }, 
            queue
        )
        
        captureSession.addOutput(videoDataOutput)
        captureSession.startRunning()
    }
    
    actual fun stopPreview() {
        captureSession.stopRunning()
    }
    
    actual fun takePhoto(): Result<ByteArray> {
        // Реализация съемки фото через AVCapturePhotoOutput
        return Result.Success(byteArrayOf())
    }
}

// Вспомогательная функция для конвертации CMSampleBuffer в ByteArray
private fun sampleBufferToByteArray(sampleBuffer: CMSampleBuffer?): ByteArray? {
    // Реализация конвертации (упрощенно)
    return null
}
```

---

## 📝 Домашнее задание (Модуль 4)

### Задача: Интегрировать нативные фичи в SkillSync

**Требования:**

1. **Кроссплатформенные карты:**
   - Реализуйте Expect/Actual для MapProvider
   - iOS: Интегрируйте MapKit с маркерами менторов
   - Android: Интегрируйте Google Maps SDK
   - Добавьте поиск ближайших менторов по геолокации

2. **AR-визитки:**
   - Реализуйте Expect/Actual для ARProvider
   - iOS: Используйте ARKit для отображения 3D-аватарки ментора
   - Android: Используйте ARCore для аналогичной функциональности
   - Добавьте сканирование плоских поверхностей

3. **Загрузка аватарок через камеру:**
   - Реализуйте Expect/Actual для CameraProvider
   - iOS: Используйте AVFoundation
   - Android: Используйте CameraX
   - Добавьте предпросмотр перед загрузкой

4. **Обработка изображений (C-interop):**
   - Подключите библиотеку libvips или ImageMagick через C-interop (только iOS/Desktop)
   - Реализуйте сжатие и оптимизацию изображений перед загрузкой

5. **Биометрическая аутентификация:**
   - iOS: LocalAuthentication (Face ID/Touch ID)
   - Android: BiometricPrompt API
   - Добавьте опцию входа по биометрии

**Критерии сдачи:**
- ✅ Карты работают на обеих платформах с маркерами
- ✅ AR-визитки отображаются корректно (если устройство поддерживает)
- ✅ Камера работает для загрузки аватарок
- ✅ Обработка изображений через C-interop (iOS/Desktop)
- ✅ Биометрический вход настроен

**Бонусные задания:**
- Интегрируйте нативный SDK для push-уведомлений (FCM/APNS)
- Добавьте поддержку Apple Pay / Google Pay через нативные API
- Реализуйте кастомный C++ алгоритм для рекомендаций навыков

---

## 💡 Советы по выполнению

1. **Начните с Expect/Actual:** Сначала создайте контракт в commonMain, потом реализуйте для каждой платформы.
2. **Тестируйте на реальном устройстве:** AR и камера не работают в эмуляторах.
3. **Изучайте документацию платформ:** Apple Developer Docs и Android Developers.
4. **Используйте существующие библиотеки:** Не пишите C-interop с нуля, если есть готовое решение.
5. **Обработайте отсутствие поддержки:** Не все устройства поддерживают AR/биометрию - добавьте graceful fallback.

---

**Следующий модуль:** В Module_05_Performance_Optimization мы изучим профилирование, оптимизацию памяти и ускорение запуска приложения.

Удачи! 🚀

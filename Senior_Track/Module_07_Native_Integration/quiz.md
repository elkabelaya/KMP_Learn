# 📝 Самопроверка: Модуль 7 — Нативные интеграции

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 10+ из 14 без подглядывания.

---

## Вопрос 1. Что из перечисленного **НЕ** описано в модуле как сценарий нативной интеграции?

- **A)** Геолокация и карты (MapKit / Google Maps SDK)
- **B)** Камера и обработка изображений (AVFoundation / CameraX)
- **C)** AR/VR (ARKit / ARCore)
- **D)** Биометрия (LocalAuthentication / BiometricPrompt)
- **E)** Расчёт скидок и валидация форм бизнес-логики

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: E**

В разделе 1 перечислены именно те 20%, которые KMP «из коробки» не покрывает: карты, камера, AR, биометрия и сторонние SDK (analytics, ads, payment). Чистая бизнес-логика (расчёты, валидация) — это как раз то, что **можно и нужно** писать в `commonMain` без всяких нативных мостов.

**Почему остальные — реальные сценарии:**
- A — MapKit на iOS и Google Maps SDK на Android
- B — AVFoundation/CoreImage против CameraX
- C — ARKit против ARCore
- D — LocalAuthentication/Secure Enclave против BiometricPrompt/StrongBox

</details>

---

## Вопрос 2. Что такое C-interop в Kotlin/Native?

- **A)** Механизм, позволяющий вызывать функции из C-библиотек напрямую из Kotlin/Native-кода
- **B)** Автоматическая генерация Kotlin-классов из Swift-исходников
- **C)** Способ запуска Kotlin-кода на JVM через JNI
- **D)** HTTP-мост, который отправляет вызовы на сервер и получает результат

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле дана схема:

```
Kotlin/Native код → C-interop layer → C-библиотека (.so/.dylib/.a)
```

То есть Kotlin/Native напрямую вызывает нативные C-функции через сгенерированный слой биндингов. Никакого JNI и никакого HTTP здесь нет.

**Почему не другие:**
- B — для Objective-C/Swift биндинги генерирует Kotlin/Native отдельно (`platform.*`), это не C-interop
- C — JNI относится к JVM, а здесь Kotlin/Native
- D — сетевой мост не участвует, вызов идёт локально в том же процессе

</details>

---

## Вопрос 3. Где в примере модуля настраивается C-interop для iOS?

- **A)** В `shared/src/iosMain/build.gradle.kts` — плагин `kotlin("native.cocoapods")`, аргументы компилятора и блок `cocoapods { pods { ... } }`
- **B)** В `shared/src/commonMain/build.gradle.kts`, чтобы библиотека была доступна везде
- **C)** Только в `Podfile` на стороне Xcode, Kotlin тут ни при чём
- **D)** В `androidMain`, потому что C-библиотеки — это всегда Android

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаг 2 модуля прямо говорит: создай `shared/src/iosMain/build.gradle.kts`, подключи `kotlin("native.cocoapods")`, передай путь к хедерам через `freeCompilerArgs` и добавь pod `VIPSDependency` в блок `cocoapods`. C-interop — платформоспецифичная вещь, поэтому он живёт в `iosMain`, а не в `commonMain`.

**Почему не другие:**
- B — в `commonMain` C-хедеры недоступны, это нарушает иерархию source set'ов
- C — `Podfile` может присутствовать, но конфигурация биндингов задаётся в Gradle
- D — примеры с libvips в модуле отнесены к iOS/Desktop, не к Android

</details>

---

## Вопрос 4. Код: работа с C-библиотекой libvips

```c
// vips.h
typedef struct {
    int width;
    int height;
    unsigned char* data;
} VipsImage;

VipsImage* vips_compress(VipsImage* image, int quality);
void vips_image_free(VipsImage* image);
```

```kotlin
val compressed = vips_compress(vipsImage, quality)
val result = compressed.data.toKotlinByteArray()
// ??? 
return result
```

Что **обязательно** нужно сделать на месте `???`, чтобы не было утечки нативной памяти?

- **A)** Вызвать `vipsImage.data.toKotlinByteArray()` ещё раз
- **B)** Вызвать `vips_image_free(compressed)`
- **C)** Вызвать `compressed.close()` — Kotlin очистит память сам
- **D)** Ничего: сборщик мусора Kotlin освободит C-память

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В эталонном коде модуля сразу после копирования данных идёт строка `vips_image_free(compressed)` с комментарием «Освобождаем память (важно для C!)». Память, выделенная внутри C-библиотеки, **не** находится под контролем GC Kotlin — её должен освободить вызов соответствующей C-функции.

**Почему не другие:**
- A — повторное чтение данных память не освободит
- C — у указателя нет такого `close()`, который вызвал бы освобождение памяти в C-библиотеке; нужно вызвать именно `vips_image_free`
- D — классическая ошибка: GC Kotlin не знает о нативных аллокациях

</details>

---

## Вопрос 5. Как Kotlin/Native даёт доступ к фреймворкам iOS вроде MapKit?

- **A)** Нужно вручную писать Swift-обёртку и подключать её через CocoaPods
- **B)** Kotlin/Native автоматически генерирует биндинги, поэтому доступен импорт `import platform.MapKit.MKMapView`
- **C)** Через `expect/actual` для каждого класса фреймворка
- **D)** Через reflection по имени класса во время выполнения

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Раздел 3 называется «Автоматическая генерация через Kotlin/Native» и показывает:

```kotlin
import platform.MapKit.MKMapView
import platform.MapKit.MKCoordinateRegion
import platform.CoreLocation.CLCoordinates
```

Kotlin/Native сам генерирует биндинги к системным фреймворкам Apple. `expect/actual` нужен для **нашей** кроссплатформенной абстракции, а не для каждого класса MapKit.

**Почему не другие:**
- A — ручная обёртка не требуется для системных фреймворков
- C — `expect/actual` используется для `MapProvider`, а не для `MKMapView`
- D — это статическая генерация на этапе компиляции, не reflection

</details>

---

## Вопрос 6. Для чего используется аннотация `@ObjCName` в примере модуля?

- **A)** Чтобы переименовать Kotlin-класс/метод в том виде, в котором он виден из Objective-C/Swift
- **B)** Чтобы поменять имя пакета в `commonMain`
- **C)** Чтобы пометить метод как `expect`
- **D)** Чтобы отключить генерацию биндингов для iOS

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Пример из модуля:

```kotlin
@ObjCName("SkillSyncImageProcessor", exact = true)
class ImageProcessor {
    @ObjCName("compressImage:quality:")
    fun compress(image: UIImage, quality: Int): UIImage { ... }
}
```

Аннотация задаёт имя, под которым Kotlin-объявление экспортируется в Objective-C/Swift (`exact = true` означает «использовать точное имя без обычных Kotlin-преобразований»).

**Почему не другие:**
- B — пакеты тут ни при чём
- C — это не `expect`/`actual`, а управление экспортируемым именем
- D — наоборот, аннотация работает **в рамках** генерации биндингов

</details>

---

## Вопрос 7. Сколько `expect`-объявлений содержит контракт `MapProvider.kt` в `commonMain`?

- **A)** 0 — только интерфейсы
- **B)** 1 — `createMapProvider()`
- **C)** 2 — `createMapProvider()` и `createMapMarker(...)`
- **D)** 4 — по одному на каждый метод интерфейса

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

В файле определены два интерфейса (`MapProvider`, `MapMarker`) и две фабрики:

```kotlin
expect fun createMapProvider(): MapProvider
expect fun createMapMarker(id: String, latitude: Double, longitude: Double, title: String): MapMarker
```

Методы `showLocation`, `addMarker`, `removeMarker`, `setCamera` объявлены обычными членами интерфейса — `expect` нужен только для платформенного создания объектов.

**Почему не другие:**
- A — фабрики явно помечены `expect`
- B — забыли вторую фабрику `createMapMarker`
- D — методы интерфейса не являются `expect`

</details>

---

## Вопрос 8. Найди ошибку (вопрос-ловушка)

```kotlin
// iosMain/platform/MapProvider.kt
actual fun createMapMarker(
    id: String, latitude: Double, longitude: Double, title: String
): MapMarker {
    return object : MapMarker {
        override val id = id
        override val latitude = latitude
        override val longitude = longitude
        override val title = title
    }
}
```

Что не так в этом фрагменте?

- **A)** Ничего, код корректный
- **B)** Нужно использовать `data class` вместо `object`
- **C)** Нужно добавить `override` перед `actual fun`
- **D)** Поля должны быть `var`, а не `val`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A** (код корректный!)

Это вопрос-ловушка. Фрагмент полностью повторяет эталон из модуля:
- `actual` подставляет реализацию `expect`-фабрики ✅
- безымянный `object : MapMarker` задаёт все четыре свойства через `override val` ✅
- значения просто копируются из аргументов, платформенных API здесь нет ✅

`setCamera` и маркеры в `commonMain` работают с этим объектом через интерфейс `MapMarker`, так что `data class` не обязателен, `override` для `actual` не нужен, а `val` полностью достаточно (маркер неизменяем).

Аналогичная реализация есть и в `androidMain` — это нормальная практика, когда общая часть тривиальна.

</details>

---

## Вопрос 9. Почему `createArProvider()` возвращает `ArProvider?` (nullable)?

- **A)** Потому что инициализация AR всегда асинхронна
- **B)** Потому что не все платформы/устройства поддерживают AR: функция возвращает `null`, если `ARWorldTrackingConfiguration.isSupported == false` или availability ARCore не `SUCCESS`
- **C)** Потому что `ArProvider` — это `sealed class`
- **D)** Потому что сессия может упасть во время `startSession()`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В контракте стоит комментарий: `// Nullable - не все платформы поддерживают AR`. В реализациях:
- iOS: `if (!ARWorldTrackingConfiguration.isSupported) return null`
- Android: `if (ArCoreApk.getInstance().checkAvailability(context) != SUCCESS) return null`

Это и есть graceful fallback, о котором говорится в советах: приложение должно корректно работать там, где AR недоступен.

**Почему не другие:**
- A — асинхронность была бы `Deferred`/`suspend`, а не `null`
- C — `sealed class` здесь `ArTrackingState`, а возвращаемый тип — интерфейс
- D — ошибки сессии отражаются в `trackingState = Lost`, а не в `null` из фабрики

</details>

---

## Вопрос 10. Код: делегат AR-сессии

```kotlin
session.delegate = object : ARSessionDelegateProtocol {
    override fun session(session: ARSession, didFailWithError: NSError?) {
        trackingState = ArTrackingState.Lost
    }
    override fun session(session: ARSession, didAddAnchors: List<ARAnchor>) {
        trackingState = ArTrackingState.Tracking
    }
}
```

Какое состояние будет выставлено, если сессия сообщит об ошибке?

- **A)** `ArTrackingState.Initializing`
- **B)** `ArTrackingState.Tracking`
- **C)** `ArTrackingState.Lost`
- **D)** `null`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

В колбэке `didFailWithError` модуль явно присваивает `trackingState = ArTrackingState.Lost`. Состояния описаны `sealed class ArTrackingState` с тремя вариантами: `Initializing`, `Tracking`, `Lost`.

**Почему не другие:**
- A — `Initializing` — стартовое значение до запуска сессии
- B — `Tracking` выставляется в `didAddAnchors`, то есть при нормальном ходе отслеживания
- D — `trackingState` имеет ненулевой тип `ArTrackingState`

</details>

---

## Вопрос 11. Какую стратегию backpressure использует Android-реализация камеры для анализа кадров?

- **A)** `ImageAnalysis.STRATEGY_BLOCK_PRODUCER`
- **B)** `ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST`
- **C)** `CameraSelector.DEFAULT_FRONT_CAMERA`
- **D)** `CameraX.STRATEGY_QUEUE_ALL`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В коде `AndroidCameraProvider.startPreview()`:

```kotlin
val imageAnalysis = ImageAnalysis.Builder()
    .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
    .build()
```

Это значит: если анализатор не успевает, старые кадры отбрасываются и остаётся только самый свежий — типичный выбор для превью камеры.

**Почему не другие:**
- A — блокировка producer'а затормозила бы саму камеру
- C — это выбор камеры, а не стратегия backpressure (в модуле, кстати, используется `DEFAULT_BACK_CAMERA`)
- D — такого варианта в примере нет

</details>

---

## Вопрос 12. Для чего в анализаторе камеры вызывается `imageProxy.close()`?

- **A)** Чтобы остановить предпросмотр
- **B)** Чтобы сохранить кадр в галерею
- **C)** Чтобы освободить буфер кадра и дать конвейеру обработать следующий
- **D)** Чтобы закрыть приложение

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

Правило CameraX: каждый полученный `ImageProxy` обязан быть закрыт. В коде сразу после `onFrameReceived(...)` идёт `imageProxy.close()` — иначе буферы закончатся и поток кадров встанет. Особенно критично это при `STRATEGY_KEEP_ONLY_LATEST`.

**Почему не другие:**
- A — для остановки есть `stopPreview()` → `cameraProvider?.unbindAll()`
- B — сохранение фото делает `takePhoto()` через `ImageCapture.OutputFileOptions`
- D — не имеет отношения к жизненному циклу приложения

</details>

---

## Вопрос 13. Какие нативные API в домашнем задании модуля предложено использовать для биометрии?

- **A)** iOS — LocalAuthentication (Face ID/Touch ID), Android — BiometricPrompt API
- **B)** iOS — ARKit, Android — ARCore
- **C)** iOS — CoreImage, Android — CameraX
- **D)** iOS — Keychain, Android — SharedPreferences

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В разделе «Сценарии использования» и в домашнем задании (пункт 5) прямо указано: iOS — `LocalAuthentication` (Face ID/Touch ID), Android — `BiometricPrompt API`. Это ещё один пример expect/actual-абстракции над разными системными API.

**Почему не другие:**
- B — это AR, отдельная задача домашки
- C — это камера и обработка изображений
- D — это про хранение данных, причём `SharedPreferences` в модуле не рассматривается как биометрический API

</details>

---

## Вопрос 14. Какой из «Советов по выполнению» модуля **НЕ** верен?

- **A)** Начинать с `expect/actual`: сначала контракт в `commonMain`, потом реализации
- **B)** Тестировать AR и камеру на реальном устройстве, потому что в эмуляторах они не работают
- **C)** Тестировать AR и камеру только в эмуляторе — это быстрее и надёжнее
- **D)** Обрабатывать отсутствие поддержки: не все устройства поддерживают AR/биометрию, нужен graceful fallback

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

Совет модуля говорит ровно наоборот: «Тестируйте на реальном устройстве: AR и камера не работают в эмуляторах». Поэтому вариант C — прямая инверсия и он неверен.

**Почему остальные верны:**
- A — совет №1 модуля
- B — совет №2 модуля
- D — совет №5 модуля про graceful fallback

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 12–14 | 🎉 Отлично! Нативная интеграция — не чёрная магия, можешь смело браться за домашку |
| 9–11 | 📖 Перечитай секции про C-interop и expect/actual для карт/AR/камеры, просмотри вопросы, где ошибся |
| 6–8 | ⚠️ Пройди модуль ещё раз. Сфокусируйся на различиях iOS (MapKit/ARKit/AVFoundation) и Android (Google Maps/ARCore/CameraX) |
| < 6 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🔜 Следующий модуль — оптимизация производительности: профилирование, память и ускорение запуска.

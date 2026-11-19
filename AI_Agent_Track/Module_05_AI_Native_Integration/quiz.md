# 📝 Самопроверка: Модуль 5 — AI в нативных интеграциях

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 11+ из 15 без подглядывания.

---

## Вопрос 1. Где должна лежать `expect`-декларация нативного сервиса?

- **A)** В `androidMain` / `iosMain` — рядом с реализацией
- **B)** В `commonMain`: единый контракт для всех платформ, а `actual`-реализации — в платформенных source set'ах
- **C)** В Swift-файле iOS-приложения
- **D)** В `AndroidManifest.xml` и `Info.plist`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Все шаблоны промптов модуля 5 начинаются с одной и той же конструкции: «expect interface из `commonMain`» → AI создаёт `actual class`, который этот интерфейс реализует. Так общий код работает с единым API, а платформенные особенности спрятаны в `androidMain` / `iosMain`.

**Почему другие не подходят:**
- A — `actual` действительно лежит там, но `expect` обязан быть в общей части
- C — Swift упоминается только как мост для платформенных extension'ов, не как место контракта
- D — манифесты нужны для разрешений, а не для объявления API

</details>

---

## Вопрос 2. Что AI должен вернуть по шаблону промпта для Android?

```text
Ты - Senior Android разработчик с 7+ лет опытом.
Контекст:
- KMP приложение с expect/actual pattern
- Target API: Android [VERSION]+ (minSdk 24)
Требования:
1. actual class implements expect interface из commonMain
2. Jetpack libraries ...
```

- **A)** Только `actual class`
- **B)** 1. Actual implementation class; 2. permissions в `AndroidManifest.xml`; 3. Permission request flow; 4. Пример использования из common code
- **C)** Swift bridge и ключи в `Info.plist`
- **D)** Только unit-тесты и README

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Шаблон из темы 5.1 прямо перечисляет четыре артефакта: actual implementation class, необходимые Android permissions в манифесте, permission request flow (если нужно) и пример вызова из common-кода. Дополнительно требуется lifecycle-aware подход, coroutines для async и `Result sealed class` для ошибок.

**Почему другие не подходят:**
- A — неполный ответ: промпт просит «Покажи полный код», включая разрешения и пример использования
- C — это iOS-сторона темы 5.2
- D — тесты и документация — тема модуля 6, не этот шаблон

</details>

---

## Вопрос 3. Что из этого **НЕ** требовалось в промпте для Android-реализации `CameraService`?

```kotlin
interface CameraService {
    suspend fun takePhoto(): Result<ByteArray>
    suspend fun startVideoRecording(): Result<Unit>
    suspend fun stopVideoRecording(): Result<String>
    fun getCameraPreview(): Flow<Bitmap>
}
```

- **A)** CameraX API (latest stable)
- **B)** Обработка `CAMERA` permission
- **C)** Front/back camera selection и валидация hardware capabilities
- **D)** CoreLocation (`CLLocationManager`) и ключи в `Info.plist`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: D**

В примере темы 5.1 к Android-реализации камеры предъявляются требования: CameraX (latest stable), `CAMERA` permission, выбор front/back камеры и проверка аппаратных возможностей. CoreLocation и `Info.plist` — это мир iOS (тема 5.2, `LocationService`).

**Почему другие не подходят:**
- A, B, C — все три пункта дословно есть в промпте

</details>

---

## Вопрос 4. Что вернёт `takePhoto()`, если разрешение `CAMERA` не выдано?

```kotlin
actual suspend fun takePhoto(): Result<ByteArray> {
    return try {
        if (!hasCameraPermission()) {
            return Result.Error(CameraError.PermissionDenied)
        }
        Result.Success(capturePhoto())
    } catch (e: Exception) {
        Result.Error(CameraError.Unknown(e.message))
    }
}
```

- **A)** `Result.Error(CameraError.PermissionDenied)`
- **B)** `Result.Success(emptyArray())`
- **C)** Брошенное `SecurityException`
- **D)** `null`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`hasCameraPermission()` проверяет `ContextCompat.checkSelfPermission(...) == PackageManager.PERMISSION_GRANTED`. Если разрешения нет, сразу возвращается `Result.Error(CameraError.PermissionDenied)`. Непредвиденные исключения ловятся `catch` и превращаются в `Result.Error(CameraError.Unknown(e.message))`.

**Почему другие не подходят:**
- B — «тихий» пустой успех скрыл бы проблему с разрешением
- C — исключение не выбрасывается наружу, оно оборачивается в `Result`
- D — метод возвращает `Result<ByteArray>`, а не nullable тип

</details>

---

## Вопрос 5. Вопрос-ловушка. Что означает «`Result sealed class` для error handling» в шаблонах модуля?

- **A)** Стандартный `kotlin.Result` из stdlib с его `success` / `failure`
- **B)** Кастомный sealed-тип `Result` с вариантами `Success` и `Error`, как в примере: `Result.Success(photo)` / `Result.Error(CameraError.PermissionDenied)`
- **C)** `java.util.Result` из JDK
- **D)** Просто аннотация `@Result` для документации

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Это ловушка на невнимательность. В сгенерированном коде используются конструкторы `Result.Success(photo)` и `Result.Error(...)`, а рядом объявлен `sealed class CameraError : Exception()` с вариантами `PermissionDenied` и `Unknown(message)`. То есть речь именно о кастомном sealed-типе результата, который шаблон требует от AI.

**Почему другие не подходят:**
- A — у stdlib `Result` нет таких конструкторов `Success/Error`, и он не является «sealed class для error handling» в смысле шаблона
- C — такого класса в JDK нет
- D — это реальный механизм обработки ошибок, а не аннотация

</details>

---

## Вопрос 6. Чем шаблон промпта для iOS отличается от шаблона для Android?

- **A)** Роль «Senior iOS разработчик 7+ лет в Swift», iOS 15.0+, разрешения через `Info.plist` + runtime requests, delegate patterns и completion handlers, disposal нативных ресурсов
- **B)** Роль «Senior Android разработчик», minSdk 24, `AndroidManifest.xml`, Lifecycle-aware компоненты
- **C)** Используется Java вместо Kotlin
- **D)** Разрешения не нужны — iOS их не запрашивает

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Тема 5.2 задаёт iOS-контекст: Kotlin Multiplatform для actual implementation, минимальный deployment target iOS 15.0+, интеграция AVFoundation/CoreLocation, `Info.plist` и runtime-запросы, delegate/completion handler'ы, coroutines, `Result sealed class` и «Proper disposal of native resources».

**Почему другие не подходят:**
- B — это шаблон темы 5.1 (Android)
- C — actual implementation пишется на Kotlin/Native, не на Java
- D — именно неправильная обработка разрешений названа отдельной ошибкой в конце модуля

</details>

---

## Вопрос 7. Что требует промпт для `LocationService` на iOS?

- **A)** CoreLocation (`CLLocationManager`), разрешения `LOCATION_ALWAYS` и `LOCATION_WHILE_USING`, background location updates и сценарий «location services disabled»
- **B)** CameraX и разрешение `CAMERA`
- **C)** ML Kit Barcode Scanning и AVFoundation
- **D)** Core ML и файл `.mlmodelc`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Промпт из примера 5.2 прямо называет CoreLocation/CLLocationManager, оба уровня разрешений (`LOCATION_ALWAYS`, `LOCATION_WHILE_USING`), поддержку фоновых обновлений (если configured) и обработку случая, когда геолокация отключена. Ожидаемый результат — полный Kotlin-код с delegate pattern и permission handling.

**Почему другие не подходят:**
- B — камера, Android
- C — QR-сканер (задание 5.1), для iOS там AVFoundation + Vision
- D — ML-интеграция (тема 5.3), не геолокация

</details>

---

## Вопрос 8. Вопрос-ловушка. «Actual-реализация для iOS должна быть на Swift, Kotlin нужен только для Android». Верно?

- **A)** Да, `iosMain` компилируется в Swift
- **B)** Нет: модуль предлагает писать actual implementation в Kotlin (`iosMain`); Swift class + Kotlin bridge упоминается только для платформенных extension'ов, когда нужного API нет в стандартных KMP-библиотеках
- **C)** Да, но только для CoreLocation
- **D)** Нет, вся бизнес-логика должна быть на Swift

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Ловушка на «нативный iOS = Swift». В теме 5.2 сказано: «Actual implementation в Kotlin (iosMain)», а ожидаемый результат — код с `CLLocationManager`, написанный на Kotlin. Swift появляется в теме 5.4 как вариант для нативного модуля (Swift class + Kotlin bridge **или** чистый Kotlin/Native).

**Почему другие не подходят:**
- A — Kotlin/Native компилируется в нативный бинарный код/фреймворк, а не «в Swift»
- C — исключений для CoreLocation нет
- D — наоборот, общая логика остаётся в Kotlin

</details>

---

## Вопрос 9. Какая связка ML-инструментов задана шаблоном темы 5.3?

- **A)** Android — ML Kit (Google); iOS — Core ML + Create ML; в `commonMain` — `predict(input): Result<Output>`, загрузка модели и обработка ошибок (missing model, invalid input)
- **B)** Android — Core ML; iOS — ML Kit
- **C)** Обе платформы используют ML Kit
- **D)** Обе платформы используют Core ML

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаблон ML-интеграции задаёт: Android → ML Kit [MODEL TYPE] API, iOS → Core ML `.mlmodelc` (модели создаются через Create ML). В `commonMain` нужен expect interface с методом inference `predict(input): Result<Output>`, загрузкой/инициализацией модели и error handling для отсутствующих моделей и невалидного входа.

**Почему другие не подходят:**
- B — подмена платформ местами
- C, D — шаблон намеренно использует разные нативные ML-стеки под каждую платформу, скрывая разницу за expect/actual

</details>

---

## Вопрос 10. Что должен возвращать expect interface для OCR по промпту примера?

```text
Android: ML Kit Text Recognition
iOS: Vision Framework VNRecognizeTextRequest
Expect interface должен возвращать:
- Recognized text (String), Text blocks с bounding boxes, Confidence scores
Обработай разные языки (multi-language support).
```

- **A)** Распознанный текст (`String`), текстовые блоки с bounding boxes, confidence scores и поддержку нескольких языков
- **B)** Только `String` с распознанным текстом
- **C)** `ByteArray` с изображением
- **D)** `Location` с координатами

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Промпт прямо перечисляет три составляющих результата: текст, блоки с прямоугольниками (bounding boxes) и оценки уверенности (confidence scores), плюс требование multi-language support. Реализации: ML Kit Text Recognition на Android и `VNRecognizeTextRequest` из Vision на iOS.

**Почему другие не подходят:**
- B — неполный контракт: теряются bounding boxes и confidence
- C — это входные данные, а не результат OCR
- D — из задания про геолокацию

</details>

---

## Вопрос 11. Как создать нативный модуль для фичи, которой нет в стандартных KMP-библиотеках?

- **A)** Android — Kotlin class с proper lifecycle management; iOS — Swift class + Kotlin bridge (или чистый Kotlin/Native); unified expect/actual API в `commonMain`
- **B)** Только на Swift — KMP здесь не участвует
- **C)** Через HTTP-запрос к серверу
- **D)** Дублировать одну и ту же реализацию в `commonMain` для всех платформ

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаблон темы 5.4 именно так и формулирует: платформенно-специфичные API, Android — Kotlin class с управлением жизненным циклом, iOS — Swift class + Kotlin bridge либо чистый Kotlin/Native, а в `commonMain` — expect/actual как единый API. Примеры фич: Deep Linking, In-App Purchases, Push Notifications.

**Почему другие не подходят:**
- B — теряется общий контракт и переиспользование
- C — сеть не заменяет локальные платформенные API (например, биометрию)
- D — платформенный API физически недоступен в `commonMain`

</details>

---

## Вопрос 12. Что требуется для unification push-уведомлений (задание 5.2)?

- **A)** FCM для Android, APNs для iOS и единый API в `commonMain`
- **B)** Один общий FCM-клиент для обеих платформ
- **C)** Только получение device tokens, без обработки сообщений
- **D)** Реализация на сервере без изменений в приложении

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Задание 5.2 требует интегрировать FCM (Android) и APNs (iOS) и собрать unified API в `commonMain`. Критерии: получение device tokens **и** обработка push-сообщений.

**Почему другие не подходят:**
- B — APNs не заменяется FCM на iOS
- C — неполное выполнение критериев
- D — клиентская часть всё равно нужна

</details>

---

## Вопрос 13. Какие стекы использовать для QR Code Scanner (задание 5.1)?

- **A)** Android — ML Kit Barcode Scanning; iOS — AVFoundation + Vision
- **B)** Android — Core ML; iOS — ML Kit
- **C)** Обе платформы — только AVFoundation
- **D)** Обе платформы — только CameraX

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В задании 5.1 прямо указано: генерировать expect/actual для QR scanning, Android — ML Kit Barcode Scanning, iOS — AVFoundation + Vision. Критерии: работает на обеих платформах и корректная обработка разрешений.

**Почему другие не подходят:**
- B — перепутаны платформы (Core ML — iOS)
- C — AVFoundation есть только на iOS
- D — CameraX — Android-сторона, и одного камеры недостаточно для распознавания кода

</details>

---

## Вопрос 14. Вопрос-ловушка. Какую версию API рекомендует модуль указывать в промпте, чтобы AI не использовал устаревшие API?

- **A)** «latest stable APIs для iOS 15+ / minSdk 24» — как в базовом шаблоне
- **B)** «latest stable APIs для iOS 17+ / Android 14+»
- **C)** «любые API, лишь бы код компилировался»
- **D)** «API образца 2018 года»

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Ловушка на смешение двух разных чисел. Базовые шаблоны задают **минимальную поддержку приложения** (iOS 15.0+, minSdk 24), а в разделе «Ошибки при нативной интеграции» против устаревших API рекомендуют явно говорить AI: «Используй latest stable APIs для iOS 17+ / Android 14+». Это именно та формулировка, которая требуется в вопросе.

**Почему другие не подходят:**
- A — это контекст шаблона, но не рекомендация из раздела ошибок
- C — именно так появляются устаревшие/нестабильные вызовы
- D — обратный эффект

</details>

---

## Вопрос 15. Какие исправления предлагаются для двух других типовых ошибок нативной интеграции — плохие разрешения и утечки памяти?

- **A)** «Включи полный permission request flow с fallback» и «Обеспечь proper cleanup и disposal of native resources»
- **B)** «Добавь `try/catch` вокруг всего» и «Увеличь версию API»
- **C)** «Игнорируй ошибки разрешений» и «Положи всё в `commonMain`»
- **D)** «Используй только Java-API» и «Отключи логирование»

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В конце модуля перечислены три ошибки и решения: устаревшие API → latest stable APIs; неправильная работа с разрешениями → полный permission request flow с fallback; утечки памяти в нативном коде → proper cleanup и disposal of native resources (что также зафиксировано в требовании iOS-шаблона «Proper disposal of native resources»).

**Почему другие не подходят:**
- B — `try/catch` не заменяет запрос разрешений, а версия API не лечит утечки
- C — прямо противоположно рекомендациям модуля
- D — не имеет отношения к причинам этих ошибок

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 11–15 | 🎉 Отлично! Нативные интеграции с AI под контролем — переходи к Модулю 6 |
| 8–10 | 📖 Перечитай темы 5.2 и 5.3, особенно permissions и expect/actual для ML |
| 5–7 | ⚠️ Пройди модуль ещё раз. Сфокусируйся на шаблонах промптов и различиях Android/iOS |
| < 5 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🔜 Дальше — [Модуль 6: AI в Production](../Module_06_AI_Production/content.md): безопасность, мониторинг, качество и рабочие процессы команды.

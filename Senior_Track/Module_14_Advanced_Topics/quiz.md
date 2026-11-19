# 📝 Самопроверка: Модуль 14 — Продвинутые темы

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 12+ из 15 без подглядывания.

---

## Вопрос 1. Что НЕ является причиной использовать C++ interop согласно модулю?

- **A)** Переиспользование существующих C/C++ библиотек: OpenCV, FFmpeg, SQLite
- **B)** Performance-critical код: обработка изображений, видео, шифрование
- **C)** Машинное обучение на C++ (TensorFlow C++, ONNX Runtime)
- **D)** Полный отказ от `expect/actual` для платформенных API

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: D**

В модуле перечислены четыре сценария: готовые C/C++ библиотеки, performance-critical код, ML (TensorFlow C++, ONNX Runtime), криптография и безопасность. C++ interop **не заменяет** `expect/actual` — в самом же примере обёртка `ImageProcessorWrapper` объявляет `expect fun createImageProcessor()` и platform-specific `actual`-реализации для Android (`System.loadLibrary` + JNI) и iOS.

**Почему другие не подходят:**
- A — это первый пункт списка сценариев
- B — второй пункт (обработка изображений и видео)
- C — третий пункт, рядом с криптографией

</details>

---

## Вопрос 2. К какому варианту interop относится этот код?

```kotlin
// Для чистого C кода
external fun printf(format: CString, vararg args: AnyVarArg)
```

- **A)** C Interop (FFI) — простой вариант
- **B)** C++ Interop (Konan) для классов и наследования
- **C)** Полная поддержка C++ interop с шаблонами
- **D)** JNI-обёртка на Java/Kotlin side

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле выделены три варианта:

| Вариант | Что даёт |
|---------|----------|
| C Interop (FFI) | простой вызов чистого C-кода через `external fun` и `CString` |
| C++ Interop (Konan) | работа с C++-библиотеками, классами, `ImageProcessor().process(image)` |
| Kotlin/Native C++ interop | классы, наследование, шаблоны (с ограничениями), автогенерация bindings из headers |

`external fun printf(...)` — это как раз первый, самый простой вариант.

**Почему другие не подходят:**
- B — для C++-классов объявлялись бы `ImageProcessor` и вызовы его методов
- C — шаблоны и наследование здесь не используются
- D — JNI появляется позже, в Android-реализации `actual`-функций

</details>

---

## Вопрос 3. Зачем C++ API обёрнут в `extern "C"`?

```cpp
extern "C" {
    void* createImageProcessor();
    void destroyImageProcessor(void* processor);
    struct ImageData* processImageC(
        void* processor, const unsigned char* pixels,
        int width, int height, int filterType
    );
    void freeImageData(struct ImageData* data);
}
```

- **A)** Чтобы получить C-совместимую linked-ABI без name mangling — так функции можно вызвать через FFI/JNI
- **B)** Чтобы включить поддержку C++17 в CMake
- **C)** Чтобы автоматически сгенерировать Kotlin-классы из C++-headers
- **D)** Чтобы использовать наследование и виртуальные методы через interop

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Имена C++-функций подвергаются mangling (в них кодируются типы и пространство имён), поэтому напрямую из FFI/JNI их вызвать нельзя. `extern "C"` отключает mangling и даёт стабильные символы — появляется плоский C-style API: создать объект (`void*`), обработать данные, освободить результат, уничтожить объект. Это альтернатива прямому C++ interop и самый надёжный путь для межъязыковой границы.

**Почему другие не подходят:**
- B — стандарт C++17 задаётся `set(CMAKE_CXX_STANDARD 17)` и `cppFlags += "-std=c++17"`
- C — генерация bindings — задача Kotlin/Native cinterop, а не `extern "C"`
- D — для наследования как раз нужен полноценный C++ interop, а не C-style API

</details>

---

## Вопрос 4. Каков верный порядок работы с нативной памятью в `ImageProcessorWrapper`?

- **A)** `createImageProcessor()` → `processImageC(...)` → `memcopy` в Kotlin `ByteArray` → `freeImageData(result)` → `dispose()` (`destroyImageProcessor`)
- **B)** `processImageC(...)` → `createImageProcessor()` → `dispose()` → `freeImageData(result)`
- **C)** `createImageProcessor()` → `dispose()` → `processImageC(...)` → `freeImageData(result)`
- **D)** `freeImageData(result)` → `processImageC(...)` → `memcopy` → `createImageProcessor()`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В коде обёртки именно такой порядок:

1. `init { processor = createImageProcessor() }` — создаём нативный объект;
2. `processImageC(processor, pixels, width, height, filterType)` — получаем `ImageData`;
3. `memcopy(output.ptrTo(0), result.pixels.data, ...)` — копируем пиксели в Kotlin `ByteArray`;
4. `freeImageData(result)` — освобождаем временный `ImageData`;
5. `dispose() { destroyImageProcessor(processor) }` — уничтожаем сам процессор.

Если пропустить шаг 4 или 5, нативная память утечёт: GC за ней не следит.

**Почему другие не подходят:**
- B — нельзя обрабатывать изображение до создания процессора
- C — `dispose()` уничтожает объект, после него вызовы невозможны
- D — frees и creates перепутаны местами

</details>

---

## Вопрос 5. 🎯 Вопрос-ловушка: что произойдёт при вызове этой `actual`-функции из `iosMain`?

```kotlin
actual fun processImageC(
    processor: CPointer<ImageProcessor>,
    pixels: ByteArray, width: Int, height: Int, filterType: Int
): ImageData {
    val result = processImageC(processor, pixels, width, height, filterType)
    return result
}
```

- **A)** Будет вызвана сгенерированная из C++ headers C-функция
- **B)** Функция рекурсивно вызовет саму себя → `StackOverflowError`
- **C)** Вернётся `null`, потому что `ImageData` не инициализирован
- **D)** Код не скомпилируется: `actual` не может вызывать `external`-функции

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Комментарий в модуле гласит «Автоматически сгенерировано из headers», но имя `processImageC` совпадает с именем самой `actual`-функции. Разрешение вызова внутри тела выбирает ближайшую по области видимости функцию — то есть саму обёртку. Получается бесконечная рекурсия.

Лечение: переименовать сгенерированную функцию (например, `processImageCNative`, как в Android-реализации) или явно импортировать её под псевдонимом.

Та же ошибка встречается в модуле в `iosMain` и для `createImageProcessor()` / `destroyImageProcessor()` / `freeImageData`, а также в `FlowExtensions.kt`:

```kotlin
fun <T> Flow<T>.debounce(delayMillis: Long): Flow<T> = this.debounce(delayMillis)
```

Внутри `this.debounce(...)` снова ссылается на саму функцию, а не на стандартный оператор Flow.

**Почему другие не подходят:**
- A — именно это и есть ловушка: «сгенерированная» функция перекрыта обёрткой
- C — тип возврата `ImageData` ненулевой, дело не в null
- D — компилятору проблема не видна, она проявляется в рантайме

</details>

---

## Вопрос 6. Для чего в `build.gradle.kts` создаётся группа `native`?

```kotlin
applyDefaultHierarchyTemplate {
    common {
        group("native") {
            withAndroidTarget()
            withIosTargets()
        }
    }
}

sourceSets {
    val nativeMain by creating { dependsOn(commonMain) }
    val androidMain by getting { dependsOn(nativeMain) }
    val iosX64Main by creating { dependsOn(nativeMain) }
}
```

- **A)** Чтобы вынести общий для Android и iOS C++/interop-код в `nativeMain`, не загрязняя `commonMain` платформенными деталями
- **B)** Чтобы полностью отказаться от `commonMain`
- **C)** Чтобы подключить OpenCV как Gradle-зависимость для JVM
- **D)** Чтобы iOS-таргеты компилировались в JVM-байткод

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`commonMain` должен оставаться полностью общим. C++-обёртки, `CPointer`, `memcopy`, `platform.posix.*` имеют смысл только для нативных target'ов, поэтому создаётся промежуточный source set `nativeMain`, который наследуют `androidMain` и все ios-наборы. Это стандартный приём иерархии KMP: общий код поднимается на самый высокий уровень, где он ещё корректен.

**Почему другие не подходят:**
- B — `commonMain` остаётся корнем иерархии, `nativeMain` от него зависит
- C — OpenCV подключается через NDK/CMake, а не как JVM-зависимость
- D — iOS компилируется в Kotlin/Native, а не в JVM

</details>

---

## Вопрос 7. Какая стратегия ML подразумевает Core ML только на iOS с fallback на Android?

- **A)** TensorFlow Lite на Android и iOS
- **B)** Core ML (iOS only) + TensorFlow Lite или серверный API на Android
- **C)** Server-side ML: приложение только отправляет запросы на Ktor-бэкенд
- **D)** ONNX Runtime на всех платформах без переделки моделей

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В модуле описаны три варианта:

1. TensorFlow Lite — Android-нативная библиотека, на iOS модель конвертируется в Core ML;
2. Core ML (iOS only) + fallback на Android: TFLite или серверный API;
3. Server-side ML на Ktor, клиент — тонкий.

Второй вариант и есть ответ: `IOSImageClassifier` работает через `VNCoreMLModel`/Vision, `AndroidImageClassifier` — через `org.tensorflow.lite.Interpreter`.

**Почему другие не подходят:**
- A — это первый вариант, без Core ML
- C — это третий вариант, без on-device ML
- D — ONNX упоминается только как сценарий C++ interop

</details>

---

## Вопрос 8. Зачем в common-коде ML-сервиса и сразу `interface`, и `expect fun createImageClassifier()`?

```kotlin
interface ImageClassifier {
    suspend fun classifyImage(imageData: ByteArray): List<ClassificationResult>
    fun isModelLoaded(): Boolean
    suspend fun loadModel(): Result<Unit>
}

expect fun createImageClassifier(): ImageClassifier
```

- **A)** Интерфейс — это 100% KMP-контракт для общей логики, а `expect/actual`-фабрика создаёт нужную реализацию (`AndroidImageClassifier` / `IOSImageClassifier`) на каждой платформе
- **B)** Интерфейс нужен только для тестов, а `expect` дублирует его по ошибке
- **C)** Так можно обойтись без фактических реализаций на платформах
- **D)** `expect` здесь заменяет dependency injection инициацию

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Общий код работает с `ImageClassifier` и не знает, что внутри TFLite или Core ML. Фабрика `createImageClassifier()` — единственная точка, где платформа «вставляет» реализацию:

- `androidMain`: `actual fun createImageClassifier() = AndroidImageClassifier()`
- `iosMain`: `actual fun createImageClassifier() = IOSImageClassifier()`

Это тот же подход, что и с `UiTestRunner` в тестах: контракт в common, создание — в platform-specific коде.

**Почему другие не подходят:**
- B — это осознанный паттерн, а не дублирование
- C — без `actual`-реализаций сборка под конкретный target упадёт
- D — фабрика решает только вопрос создания объекта, а не весь DI

</details>

---

## Вопрос 9. Что в коде ML-сервиса модуля всё ещё является заглушкой и требует реальной реализации?

- **A)** `loadModelFile` (TODO: загрузка из assets с Context), `prepareInput` (упрощённая нормализация/ресайз) и `toCGImage()` (TODO на iOS)
- **B)** `isModelLoaded()` и `parseResults()`
- **C)** `classifyImage()` целиком — он написан только как комментарий
- **D)** Ничего: код полностью готов к production

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В коде прямо стоят TODO и комментарии:

```kotlin
private fun loadModelFile(fileName: String): MappedByteBuffer {
    TODO("Implement model loading from assets")
}

private fun prepareInput(imageData: ByteArray): ByteBuffer {
    // Нормализация и ресайз до 224x224 ... Для упрощения - заглушка
}

fun ByteArray.toCGImage(): CGImage {
    TODO("Implement ByteArray to CGImage conversion")
}
```

`parseResults`, напротив, уже читает топ-5 confidence и сортирует результаты по убыванию.

**Почему другие не подходят:**
- B — `:isModelLoaded()` — простая проверка флага, `parseResults` реализован
- C — `classifyImage` написан, он использует `require(isModelLoaded)`, `Interpreter.run` и парсинг
- D — три заглушки делают код учебным каркасом, а не готовым сервисом

</details>

---

## Вопрос 10. Как iOS-классификатор получает результаты предсказания?

- **A)** Запрашивает `count: 5` через `VNImageClassificationRequest`, выполняет `VNImageRequestHandler.perform(...)`, читает `NSArray` и берёт `identifier` + `confidence` из `VNClassificationObjectObservation`
- **B)** Отправляет изображение на Ktor-сервер и получает JSON с метками
- **C)** Вызывает `Interpreter.run(inputBuffer, outputBuffer)`, как Android
- **D)** Использует `ByteBuffer.allocateFloat(1000)` и читает 1000 классов

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`IOSImageClassifier.loadModel()` создаёт `VNCoreMLModel.create(from: mlModel)`, а `classifyImage()` конвертирует `ByteArray` в `CGImage`, ставит `count: 5`, выполняет запрос через `VNImageRequestHandler` и мапит первые 5 observation в `ClassificationResult(label = result.identifier, confidence = result.confidence.toFloat())`.

**Почему другие не подходят:**
- B — это третий вариант стратегии (server-side ML), здесь он не используется
- C — `Interpreter` — это TensorFlow Lite, Android-ветка
- D — это код Android-заглушки для 1000 классов, не iOS-ветка

</details>

---

## Вопрос 11. Почему `Counter` содержит race condition и как его чинит Actor?

```kotlin
class Counter {
    var count = 0
    fun increment() { count++ } // Не атомарно!
}

class CounterActor {
    private val mailbox = Channel<CounterCommand>(Channel.UNLIMITED)
    private var count = 0
    init {
        CoroutineScope(Dispatchers.Default).launch { processCommands() }
    }
    // Increment и GetCount(replyChannel) обрабатываются в processCommands()
}
```

- **A)** `count++` — это read-modify-write сразу из нескольких потоков; Actor сериализует команды через `mailbox`, и всё изменение состояния происходит в одной корутине
- **B)** Проблема в отсутствии `@Synchronized`: достаточно добавить её к `increment()`
- **C)** Проблема в `Dispatchers.Default`; с `Dispatchers.Main` гонка исчезнет сама
- **D)** Actor здесь не нужен: достаточно сделать `count` `volatile`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`count++` состоит из чтения, прибавления и записи — между ними поток может быть вытеснен, и приращения потеряются. В Actor-подходе внешние вызовы лишь кладут команду в `Channel`, а состояние меняет единственная корутина в `for (command in mailbox)`. Для запроса значения используется reply-канал: `GetCount(replyChannel)` → `replyChannel.send(count)` → `replyChannel.receive()`.

Базовый `Actor<Command>` в модуле устроен так же: `mailbox: Channel<Command>(Channel.UNLIMITED)`, `send()` через `trySend`, `dispose()` через `scope.cancel()`.

**Почему другие не подходят:**
- B — `@Synchronized` блокирует поток, это грубое решение и оно не описано модулем; Actor — рекомендуемый паттерн
- C — смена dispatcher не убирает саму природу read-modify-write
- D — `volatile` даёт видимость, но не атомарность `++`

</details>

---

## Вопрос 12. В чём ошибка метода `sendAndWait` из базового `Actor`?

```kotlin
fun send(command: Command) {
    mailbox.trySend(command)
}

suspend fun sendAndWait(command: Command): Boolean {
    return mailbox.send(command)
}
```

- **A)** Нельзя вызывать `send` из `suspend`-функции
- **B)** `Channel.send()` возвращает `Unit`, а не `Boolean` — код не скомпилируется; для булева результата нужен `trySend` (или `send` + `return true`)
- **C)** Команды нужно отправлять только в `processCommands()`
- **D)** Перед отправкой надо вызвать `mailbox.close()`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`Channel.send(element)` — suspend-функция с результатом `Unit`: она либо отправит элемент, либо приостановится/упадёт. Она не сообщает `true/false`. `trySend` — как раз неблокирующий вариант, возвращающий результат отправки (в версиях coroutines он возвращал `Boolean`, позже — `ChannelResult`). Поэтому корректно:

```kotlin
suspend fun sendAndWait(command: Command) {
    mailbox.send(command)
}
```

или вернуть `true` после успешного `send`.

**Почему другие не подходят:**
- A — `suspend`-функция как раз и может вызывать `send`
- C — `processCommands()` только читает из mailbox
- D — `close()` вызывается в `finally` при завершении обработки, а не перед отправкой

</details>

---

## Вопрос 13. При `retry(times = 3)`, если каждая попытка бросает исключение, что произойдёт на последней попытке?

```kotlin
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
```

- **A)** Исходное исключение будет переброшено наружу (`throw e`)
- **B)** Будет выброшено `RuntimeException("Max retries exceeded")`
- **C)** Функция вернёт `null`
- **D)** Повторы продолжатся бесконечно

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Условие `attempt == times - 1` означает: на последней итерации (`attempt = 2` при `times = 3`) мы больше не спим, а сразу перебрасываем пойманное исключение. Строка `throw RuntimeException("Max retries exceeded")` после `repeat` фактически недостижима — `repeat` всегда либо возвращается из блока, либо бросает.

Задержки между попытками задаются suspend-лямбдой `delay(attempt)` с экспоненциальным ростом; обрати внимание, что индекс `attempt` в `repeat` начинается с нуля, поэтому комментарий «500ms, 1s, 2s» — приблизительная иллюстрация идеи backoff, а не точный расчёт.

**Почему другие не подходят:**
- B — эта строка недостижима при стандартном сценарии
- C — тип возврата `T` ненулевой, `null` здесь не возвращается
- D — `repeat(times)` ограничивает число попыток

</details>

---

## Вопрос 14. Сервис после 5 неудач за 60 секунд перестаёт дёргать API и бросает `CircuitBreakerOpenException`. Какой паттерн описан?

- **A)** Circuit Breaker
- **B)** Rate Limiter
- **C)** Bulkhead
- **D)** Cache Flow с TTL

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`CircuitBreaker(failureThreshold = 5, resetTimeout = 60_000)` считает ошибки, при достижении порога открывается (`isOpen = true`) и бросает `CircuitBreakerOpenException`, не давая запросам идти дальше. Через `resetTimeout` состояние сбрасывается. Это защита от каскадных отказов.

Остальные паттерны из модуля решают другие задачи:

| Паттерн | Назначение |
|---------|-----------|
| `RateLimiter(rate, windowSize)` | не больше N запросов в секунду, ожидание до освобождения окна |
| `Bulkhead(maxConcurrent)` | ограничение числа одновременных операций |
| `cacheWithTtl(ttl)` | `stateIn` с `SharingStarted.WhileSubscribed(ttl)` |

**Почему другие не подходят:**
- B — RateLimiter не считает ошибки, он считает частоту вызовов
- C — Bulkhead ограничивает параллелизм, а не последствия ошибок
- D — TTL кэша здесь ни при чём

</details>

---

## Вопрос 15. Что НЕ соответствует production-конфигурации и критериям сдачи модуля?

- **A)** `isMinifyEnabled = true` и `isShrinkResources = true` для release-сборки
- **B)** `abiFilters` содержит `armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64`
- **C)** `kotlin.native.cacheKind=none` включают, чтобы ускорить локальную разработку
- **D)** Критерий сдачи: размер APK < 30MB

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

В `gradle.properties` модуля написано:

```properties
kotlin.native.cacheKind=none # Отключаем cache для CI
```

То есть это настройка именно для CI, а не для ускорения локальной сборки — как раз наоборот, локально кэш обычно полезен. Всё остальное верно: release-сборка включает minify и shrink resources, ABI-фильтры перечислены в модуле, а «размер APK < 30MB» — один из критериев сдачи ДЗ (наряду с crash reporting и analytics).

**Почему другие не подходят:**
- A — это прямо прописано в `android { buildTypes { release { ... } } }`
- B — ровно такой список `abiFilters` приведён в конфигурации
- D — это один из пунктов «Критерии сдачи»

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 13–15 | 🎉 Senior+ уровень: C++ interop, ML и конкурентность у тебя под контролем |
| 10–12 | 📖 Перечитай секции про expect/actual-обёртки и паттерны конкурентности |
| 6–9 | ⚠️ Пройди модуль ещё раз; начни с диаграммы source set'ов и жизненного цикла нативной памяти |
| < 6 | 🆘 Давай разберём вместе: выбери один фрагмент кода и разбери его по строкам |

---

> 🎓 Это финальный модуль Senior Track. Поздравляю — дальше портфолио, реальный production и вклад в open-source KMP. 🚀

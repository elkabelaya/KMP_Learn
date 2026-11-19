# 📝 Самопроверка: Модуль 1 — Advanced KMP Fundamentals

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 10+ из 14 без подглядывания.

---

## Вопрос 1. Что даёт компиляция KMP-проекта согласно схеме процесса из модуля?

- **A)** Один общий JAR, который потом конвертируется и в APK, и в IPA
- **B)** JVM → JAR-файлы, JS/JSR → JavaScript, Native → бинарник; далее Android / Web / iOS приложения
- **C)** Только Kotlin/Native-бинарник для всех платформ сразу
- **D)** Исходники на Swift и Dart, которые компилируются отдельно

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В модуле схема компиляции выглядит так:

```
Source Code (Kotlin) → [Compilation] → JVM Target (JAR) | JS/JSR Target (JavaScript) | Native Target (Binary)
                                            ↓                    ↓                         ↓
                                      Android App           Web App                  iOS App
```

Один и тот же Kotlin-код проходит через разные бэкенды компилятора, и каждый выдаёт артефакт своего формата.

**Почему другие не подходят:**
- A — единого JAR для всех платформ не бывает; у каждого target'а свой артефакт
- C — Native — лишь один из трёх бэкендов, JVM и JS никуда не деваются
- D — Kotlin не компилируется в Swift/Dart как промежуточный язык

</details>

---

## Вопрос 2. Какая цепочка source set'ов соответствует иерархии из модуля?

- **A)** `commonMain → jvmMain → androidMain / jvmDesktop`; `commonMain → nativeMain → iosX64 / iosArm64`
- **B)** `androidMain → jvmMain → commonMain` (depend'ы идут снизу вверх)
- **C)** `commonMain → androidMain → jvmMain → nativeMain`
- **D)** `iosX64 → nativeMain → commonMain → jvmMain` — всё в одну цепочку

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Иерархия из модуля:

```
commonMain
   ├── jvmMain    → androidMain, jvmDesktop
   ├── jsMain     → jsIr, jsJs
   └── nativeMain → iosX64, iosArm64 (и др. native-таргеты)
```

`commonMain` — база для всех, промежуточные `jvmMain`/`jsMain`/`nativeMain` группируют платформы по бэкенду, а листовые source set'ы — конкретные таргеты.

**Почему не другие:**
- B — направление перевёрнуто: ребёнок зависит от родителя, а не наоборот
- C — `androidMain` не является родителем для `jvmMain`
- D — `iosX64` — лист, а не корень; `nativeMain` не «внутри» `jvmMain`

</details>

---

## Вопрос 3. Найди корректное объявление expect-класса для PlatformStorage (Pattern 1):

```kotlin
// commonMain
expect open class PlatformStorage {
    suspend fun save(key: String, value: String): Boolean
    suspend fun load(key: String): String?
    suspend fun delete(key: String): Boolean
}
```

- **A)** Именно так — у expect-объявления есть сигнатуры, но нет реализаций
- **B)** Ошибка: нужно добавить тела с `TODO()` прямо в commonMain
- **C)** Ошибка: в commonMain нужно писать `actual open class`
- **D)** Ошибка: `suspend`-функции нельзя объявлять в expect-классе

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Так и показано в модуле: `expect open class PlatformStorage` задаёт контракт — имя, модификатор `open` и `suspend`-сигнатуры без тел. Реализации (`actual class PlatformStorage actual constructor()`) находятся в `androidMain` (SharedPreferences) и `iosMain` (NSUserDefaults).

**Почему не другие:**
- B — тела добавляются только в `actual`, в commonMain их быть не должно
- C — `actual` пишется в платформенных source set'ах, а не в commonMain
- D — `suspend` в expect-объявлениях полностью допустим (см. сигнатуры `save/load/delete`)

</details>

---

## Вопрос 4. Вопрос-ловушка. Сравни реализации из модуля:

```kotlin
// androidMain
actual suspend fun save(key: String, value: String): Boolean = withContext(Dispatchers.IO) {
    prefs.edit().putString(key, value).apply()
    true
}

// iosMain
actual suspend fun save(key: String, value: String): Boolean = withContext(Dispatchers.Default) {
    defaults.setObject(value, key)
    true
}
```

Что здесь верно?

- **A)** Ошибка: на iOS тоже обязательно нужен `Dispatchers.IO`
- **B)** Ошибка: все `actual`-реализации должны использовать один и тот же Dispatcher
- **C)** Всё корректно: контракт (`suspend`, сигнатура, тип) совпадает, а платформенные API и их dispatchers вправе отличаться
- **D)** `withContext` здесь лишний — `suspend` сам переключает поток

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

Это ловушка на кажущееся несоответствие. Разница в коде ожидаема и есть смысл expect/actual:

- Android использует `SharedPreferences` и `withContext(Dispatchers.IO)`
- iOS использует `NSUserDefaults` и `withContext(Dispatchers.Default)`
- При этом **сигнатура, `suspend` и возвращаемый тип `Boolean` одинаковы** — commonMain вызывает единый контракт

**Почему не другие:**
- A/B — компилятор требует совпадения сигнатуры, а не совпадения внутреннего dispatcher'а
- D — `suspend` сам по себе не задаёт поток; `withContext` явно выбирает нужный контекст

</details>

---

## Вопрос 5. Pattern 2 (expect sealed class). Где объявляются наследники вроде `Fingerprint` / `FaceID`?

- **A)** В commonMain рядом с `expect sealed class PlatformBiometricType`
- **B)** Только в platform-specific `actual`-реализациях (`androidMain` / `iosMain`)
- **C)** В `companion object` в commonMain
- **D)** В отдельном source set `commonSealedMain`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В commonMain объявляются только `expect sealed class PlatformBiometricType` и `companion object` с объявлением `fun getAvailableTypes(): List<PlatformBiometricType>` без тела. А конкретные наследники живут в actual-классах:

- `androidMain`: `actual object Fingerprint`, `actual object FaceRecognition`
- `iosMain`: `actual object FaceID`, `actual object TouchID`

Метод `getAvailableTypes()` возвращает разные списки для каждой платформы.

**Почему не другие:**
- A/C — commonMain не знает про `Fingerprint` или `FaceID`; это платформенные типы
- D — такого source set'а нет

</details>

---

## Вопрос 6. Pattern 3 (extension-функции). Какая реализация относится к `androidMain`?

```kotlin
// commonMain
expect fun String.platformEncode(): String
expect fun String.platformDecode(): String
```

- **A)** `android.util.Base64.encodeToString(toByteArray(), android.util.Base64.NO_WRAP)`
- **B)** `toByteArray().platform.toBase64String()`
- **C)** `NSUserDefaults.standardUserDefaults.stringForKey(this)`
- **D)** `String(kotlinx.cinterop.byteArrayOf(*toByteArray().platform.fromBase64String()))`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В `androidMain` кодирование идёт через `android.util.Base64` с флагом `NO_WRAP`, декодирование — через `android.util.Base64.decode(...)`.

**Почему не другие:**
- B и D — это iOS-реализации (`toBase64String()` / `fromBase64String()` через `kotlinx.cinterop`)
- C — вообще другой API (UserDefaults), к Base64 отношения не имеет

**Запомни:** в commonMain только `expect fun String.platformEncode(): String`, весь платформенный код — в `actual`.

</details>

---

## Вопрос 7. Для чего в `build.gradle.kts` используются `group("mobile")` и `group("desktop")` внутри `applyDefaultHierarchyTemplate`?

- **A)** Чтобы создать custom source sets `mobileMain` / `desktopMain`, от которых наследуют конкретные платформы
- **B)** Чтобы удалить лишние таргеты из сборки
- **C)** Чтобы объединить все таргеты обратно в один `commonMain`
- **D)** Чтобы настроить публикацию библиотеки

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Группы задают кастомную иерархию:

```
common
 ├── group("mobile")  → withAndroidTarget() + withIosTargets()  → mobileMain
 └── group("desktop") → withMacosTargets() + linuxX64() + mingwX64() → desktopMain
```

Далее `androidMain`/`ios*Main` зависят от `mobileMain`, а `macosX64Main`/`macosArm64Main`/`linuxX64Main`/`mingwX64Main` — от `desktopMain`. general-код для macOS + Linux + Windows пишется один раз.

**Почему не другие:**
- B — таргеты (`androidTarget`, `iosArm64`, `linuxX64`, `mingwX64`) остаются на месте
- C — наоборот, создаются дополнительные уровни, а не упраздняются
- D — публикация (`publishLibraryVariants`) настраивается отдельно в `androidTarget`

</details>

---

## Вопрос 8. Какая запись задаёт правильное направление зависимостей в кастомной иерархии?

- **A)** `commonMain.dependsOn(mobileMain)`
- **B)** `mobileMain.dependsOn(commonMain)` и `androidMain.dependsOn(mobileMain)`
- **C)** `androidMain.dependsOn(commonMain)` — и больше ничего
- **D)** `mobileMain.dependsOn(androidMain)`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В модуле именно так:

```kotlin
val mobileMain by creating { dependsOn(commonMain) }
val androidMain by getting  { dependsOn(mobileMain) }
val desktopMain by creating { dependsOn(commonMain) }
val macosX64Main by creating { dependsOn(desktopMain) }
```

Ребёнок видит всё из родителя: `androidMain` получает и `mobileMain`, и `commonMain`.

**Почему не другие:**
- A и D — перевёрнутое направление, `commonMain` ничего не должен «знать» про мобильный код
- C — потерян промежуточный слой `mobileMain`, общий для Android+iOS код некуда положить

</details>

---

## Вопрос 9. Какая задача НЕ перечислена в `dependsOn` кастомного task'а `buildAllPlatforms`?

- **A)** `assembleRelease`
- **B)** `iosX64Binaries`
- **C)** `iosArm64Binaries`
- **D)** `iosSimulatorArm64Binaries`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: D**

В модуле task'а зависит от: `assembleRelease`, `iosX64Binaries`, `iosArm64Binaries`, `macosX64Binaries`, `linuxX64Binaries`.

`iosSimulatorArm64Binaries` в списке **отсутствует** — хотя сам таргет `iosSimulatorArm64()` в конфигурации объявлен. Точно так же не попали `macosArm64Binaries` и `mingwX64Binaries`.

**Почему не другие:**
- A, B, C — все три явно есть в `dependsOn(...)`

> ⚠️ Практический вывод: кастомный task нужно сверять со списком таргетов — легко забыть симуляторную сборку.

</details>

---

## Вопрос 10. Что делает этот фрагмент конфигурации product flavors?

```kotlin
productFlavors.all { flavor ->
    kotlin.sourceSets.create("${flavor.name}Common") {
        dependsOn(kotlin.sourceSets.getByName("commonMain"))
    }
}
```

- **A)** Создаёт по KMP source set на каждый flavor (`developmentCommon`, `stagingCommon`, `productionCommon`), зависящий от `commonMain`
- **B)** Удаляет product flavors из Android-конфигурации
- **C)** Меняет `applicationId` для каждого окружения
- **D)** Регистрирует задачу `buildAllPlatforms`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Это «мост» между Android-флейворами и KMP: для каждого flavor'а (`development`, `staging`, `production` с их `applicationId`) создаётся одноимённый source set вида `<flavor>Common`, который наследует общий код от `commonMain`. Туда кладётся код, специфичный для окружения.

**Почему не другие:**
- B — flavors наоборот, создаются выше через `create("development") { ... }`
- C — `applicationId` задаётся внутри `create(...)`, а не здесь
- D — кастомный task описывается отдельно через `tasks.register("buildAllPlatforms")`

</details>

---

## Вопрос 11. Вопрос-ловушка. Что верно про этот фрагмент бенчмарка?

```kotlin
val startTime = TimeSource.Monotonic.markNow()
repeat(iterations) { block() }
val endTime = TimeSource.Monotonic.markNow()
val duration = startTime.elapsedNow()
```

- **A)** `duration` считается как разница `endTime - startTime`
- **B)** Переменная `endTime` не используется: длительность берётся через `startTime.elapsedNow()`, это лишняя строка
- **C)** `elapsedNow()` нельзя вызывать у результата `markNow()`
- **D)** В `duration` попадает и warm-up из `repeat(100) { block() }`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Ещё одна ловушка на внимание. `endTime` создаётся, но нигде не читается — замер идёт через `startTime.elapsedNow()`. По коду из модуля именно так:

```kotlin
val endTime = TimeSource.Monotonic.markNow()
val duration = startTime.elapsedNow()
```

**Почему не другие:**
- A — вычитания `endTime - startTime` в коде нет
- C — `markNow()` возвращает `TimeStamp`, у которого есть `elapsedNow()`
- D — warm-up `repeat(100) { block() }` выполняется **до** `markNow()`, поэтому в замер не входит (в этом и есть его назначение)

</details>

---

## Вопрос 12. Зачем перед основным замером в `benchmarkBlock` делается `repeat(100) { block() }`?

- **A)** Просто проверить, что `block` не бросает исключений
- **B)** Warm-up: прогреть код, чтобы исключить разовые накладные расходы и получить стабильные цифры
- **C)** Чтобы посчитать `throughput`
- **D)** Чтобы заполнить `ObjectPool`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В модуле это явно помечено комментариями:

```kotlin
// Warm-up runs
repeat(100) { block() }
// Actual benchmark
val startTime = TimeSource.Monotonic.markNow()
```

Прогрев убирает влияние JIT/первого запуска из итогового времени.

**Почему не другие:**
- A — исключения всё равно пробьются и в основном цикле
- C — `throughput = iterations / duration.inSeconds` вычисляется уже после замера
- D — `ObjectPool` — отдельная техника оптимизации, к бенчмарку не относится

</details>

---

## Вопрос 13. Что делает `cacheIn(CoroutineScope(SupervisorJob()))` в `optimizedProcessing()`?

- **A)** Меняет dispatcher на `Dispatchers.IO`
- **B)** Кэширует значения Flow, чтобы несколько подписчиков не пересчитывали их заново
- **C)** Удаляет дубликаты из потока
- **D)** Гарантирует отмену всего потока при первой же ошибке

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Цепочка из модуля:

```kotlin
fun <T> Flow<T>.optimizedProcessing(): Flow<T> = this
    .flowOn(Dispatchers.Default)                    // обработка в фоне
    .cacheIn(CoroutineScope(SupervisorJob()))       // кэш для нескольких подписчиков
    .onStart { /* Pre-warming logic */ }
```

`cacheIn` как раз и предназначен для раздельного кэширования между подписчиками.

**Почему не другие:**
- A — за dispatcher отвечает `flowOn(Dispatchers.Default)`
- C — для этого есть `distinctUntilChanged()`
- D — `SupervisorJob` как раз разделяет ошибки дочерних корутин, а не «отменяет всё»

</details>

---

## Вопрос 14. В `ObjectPool<T>` (ArrayDeque + synchronized) что происходит в `acquire()`, если пул пуст?

- **A)** Бросается исключение «пул пуст»
- **B)** Возвращается `null`
- **C)** Создаётся новый объект через `factory()`
- **D)** Поток блокируется в ожидании, пока кто-нибудь не вызовет `release`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

Код из модуля:

```kotlin
fun acquire(): T = synchronized(pool) {
    if (pool.isNotEmpty()) pool.removeLast() else factory()
}

fun release(obj: T) = synchronized(pool) {
    if (pool.size < maxSize) pool.addLast(obj)   // maxSize = 100
}
```

Объект создаётся лениво, `release` возвращает его обратно, пока не достигнут `maxSize`. `synchronized` защищает `ArrayDeque` при одновременном доступе из разных потоков.

**Почему не другие:**
- A/B — в коде нет ни исключения, ни `null` (тип `T : Any`)
- D — блокировки ожидания нет: `synchronized` только сериализует доступ к очереди

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 12–14 | 🎉 Отлично! Internals, expect/actual-паттерны и оптимизации — твоя сильная сторона |
| 9–11 | 📖 Перечитай секции про `applyDefaultHierarchyTemplate` и performance-оптимизации |
| 5–8 | ⚠️ Пройди модуль ещё раз: сфокусируйся на expect/actual-паттернах и кастомной иерархии source set'ов |
| < 5 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🔜 Следующий модуль Senior Track — продвинутая архитектура и модульность (DDD, feature-модули, Version Catalogs).

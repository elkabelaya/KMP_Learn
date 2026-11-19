# 📝 Самопроверка: Модуль 1

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 8+ из 12 без подглядывания.

---

## Вопрос 1. Что такое `commonMain`?

- **A)** Папка с кодом, который компилируется только для JVM
- **B)** Источник с кодом, доступным для всех целевых платформ (Android, iOS, Desktop)
- **C)** Папка с тестами
- **D)** Alias для `src/main`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`commonMain` — это source set (набор исходников), который компилируется для **каждой** целевой платформы, указанной в `kotlin { }` блоке. Если у тебя Android + iOS — код из `commonMain` будет скомпилирован и в Android APK, и в iOS framework.

**Почему другие не подходят:**
- A — не «только JVM», а для **всех** target'ов
- C — тесты лежат в `commonTest`
- D — `src/main` это Android-specific; `commonMain` — кроссплатформенный аналог

</details>

---

## Вопрос 2. В какой source set положить код работы с файловой системой (`File`, `NSFileManager`)?

- **A)** `commonMain` — это же просто работа с файлами, логика общая
- **B)** В `commonMain` объявить `expect fun readFile(path: String): String`, а в `androidMain` и `iosMain` — `actual`
- **C)** Дублировать код в `androidMain` и `iosMain` отдельно, без commonMain
- **D)** В `androidMain`, а для iOS написать на Swift отдельно

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`java.io.File` и `NSFileManager` — это платформенные API. Но **логика** (что читать, что делать с содержимым) — общая. Поэтому:
- В `commonMain` объявляем контракт: `expect fun readFile(path: String): String`
- В `androidMain`: `actual fun readFile(path: String): String = File(path).readText()`
- В `iosMain`: `actual fun readFile(path: String): String = ...` (через `NSFileManager` или `Platform`)

**Почему не другие:**
- A — `File` не существует на iOS (только `kotlin.io` с ограничениями)
- C — теряем переиспользование, пишем логику дважды
- D — тот же минус: логика дублируется

</details>

---

## Вопрос 3. Что произойдёт, если в `commonMain` объявить `expect fun foo()`, а `actual` написать только для Android?

- **A)** Компилятор для Android скомпилируется, для iOS — упадёт с ошибкой
- **B)** Компилятор скомпилирует всё, но на iOS `foo()` будет возвращать `null`
- **C)** Gradle не даст создать проект
- **D)** Kotlin автоматически сгенерирует пустой `actual` для iOS

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Kotlin-компилятор **требуется**, чтобы для каждого target'а был соответствующий `actual`. Если хотя бы для одного target'а `actual` отсутствует — компиляция именно этого target'а упадёт:

```
e: Expected an actual declaration for 'foo(): Unit'
```

Проект для Android при этом может скомпилироваться нормально (если ты собираешь только Android). Но полный билд (Android + iOS) сломается.

**Почему не другие:**
- B — Kotlin не делает «тихие» фоллбэки. Нет `actual` — нет функции
- C — Gradle не знает о `expect/actual` на уровне создания проекта
- D — автогенерация не существует (и слава богу, иначе были бы баги)

</details>

---

## Вопрос 4. Найди ошибку:

```kotlin
// commonMain/platform/Platform.kt
package com.ecotrack.platform
expect fun getPlatformName(): String
```

```kotlin
// androidMain/platform/Platform.kt
package com.ecotrack.platform
actual fun getPlatformName(): String = "Android"
```

```kotlin
// iosMain/platform/Platform.kt
package com.ecotrack.platform
actual fun getPlatformName(): String = "IOS"
```

Что не так?

- **A)** Ничего, код корректный
- **B)** Пакет в iosMain должен быть другим
- **C)** Функция возвращает разные типы
- **D)** В androidMain пропущен `override`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A** (код корректный!)

Это вопрос-ловушка. Всё в порядке:
- Пакеты совпадают ✅
- Имена функций совпадают ✅
- Возвратный тип совпадает (`String`) ✅
- `override` не нужен — `actual` это не override, это «заполнение» expect ✅

Разница в значении (`"Android"` vs `"IOS"`) — это и есть **смысл** expect/actual: разные реализации на разных платформах.

</details>

---

## Вопрос 5. Что из перечисленного **можно** положить в `commonMain`?

1. `data class User(val id: Long, val name: String)`
2. `fun sendNotification(title: String)` с использованием `NotificationManager`
3. Compose-функция `@Composable fun UserCard(user: User)`
4. `fun getSharedPreferences(): SharedPreferences`

- **A)** 1, 2, 3, 4
- **B)** 1, 3
- **C)** 1, 2, 3
- **D)** 2, 4

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B** (1 и 3)

| # | Можно? | Почему |
|---|:------:|--------|
| 1 | ✅ | `data class` — чистый Kotlin, нет платформенных зависимостей |
| 2 | ❌ | `NotificationManager` — Android API. Нужно через `expect/actual` |
| 3 | ✅ | Compose Multiplatform компилируется под все платформы |
| 4 | ❌ | `SharedPreferences` — Android API |

**Запомни правило:** если в импорте есть `android.*`, `kotlinx.cinterop.*`, `NS*` (Foundation) — это не `commonMain`.

</details>

---

## Вопрос 6. Зачем нужны `iosX64()`, `iosArm64()` и `iosSimulatorArm64()`?

- **A)** Это три разных языка, на которые компилируется Kotlin
- **B)** Это три аппаратные архитектуры: Intel Mac, Apple Silicon (реальные устройства), Apple Silicon (симулятор)
- **C)** Это три версии iOS
- **D)** Это опечатки, достаточно одного `ios()`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

| Target | Что это |
|--------|---------|
| `iosX64` | Симулятор на Mac с Intel-чипом |
| `iosArm64` | Реальные iPhone/iPad на Apple Silicon |
| `iosSimulatorArm64` | Симулятор на Mac с Apple Silicon (M1/M2/M3/M4) |

Почему три? Потому что **бинарный формат** (Mach-O) и **ABI** отличаются. Код, скомпилированный под `iosArm64`, не запустится на Intel Mac. И наоборот.

С 2023 года практически все Mac — Apple Silicon, поэтому `iosX64` используется всё реже, но для совместимости его всё ещё указывают.

**Почему не другие:**
- A — Kotlin компилируется в Kotlin/Native (не «три языка»)
- C — версия iOS не влияет на target
- D — `ios()` как единый target не существует (был в старых версиях, убран)

</details>

---

## Вопрос 7. Как называется файл/обёртка, через которую Compose UI вставляется в iOS-приложение?

- **A)** `ComposeActivity`
- **B)** `ComposeUIViewController`
- **C)** `ComposeView`
- **D)** `KotlinNativeContainer`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

| Платформа | Обёртка |
|-----------|---------|
| Android | `ComponentActivity` + `setContent { }` |
| iOS | `ComposeUIViewController` |
| Desktop | `composeApplication { Window { ... } }` |

`ComposeUIViewController` — это мост: Kotlin создаёт `UIViewController`, а внутри него рендерится Compose-дерево. В Swift-коде ты просто импортируешь этот контроллер:

```swift
import shared

struct ContentView: View {
    var body: some View {
        MainViewController()
    }
}
```

**Почему не другие:**
- A — `ComponentActivity` это Android
- C — `ComposeView` — это для встраивания Compose в нативный Android-View (не наш случай)
- D — такого класса нет

</details>

---

## Вопрос 8. Что делает `expect` / `actual` на уровне байт-кода?

- **A)** Генерирует мост (bridge) на платформенном языке, который делегирует вызов
- **B)** Просто переименовывает функцию
- **C)** Создаёт HTTP-запрос на сервер с реализацией
- **D)** Ничего не делает — это просто аннотация для документации

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Под капотом:
- **Kotlin/Native** (для iOS): `expect` функция создаётся как **неабстрактная функция**, а `actual` — как её реализация. При компиляции для iOS генерируется Swift-мост, который вызывает правильную реализацию.
- **Kotlin/JVM** (для Android): `expect`/`actual` компилируется как обычная функция. На JVM это работает через **inline classes** и **multiplatform plugin** — компилятор подставляет правильную реализацию.

Ключевое: из `commonMain` ты вызываешь `getPlatformName()` как обычную функцию. Магия происходит **на этапе компиляции**, а не в рантайме.

**Почему не другие:**
- B — не переименование, а именно подстановка реализации
- C — никакого HTTP
- D — нет, это реальный механизм, а не просто «подсветка»

</details>

---

## Вопрос 9. У тебя KMP-проект с target'ами Android + iOS + Desktop. В `commonMain` есть:

```kotlin
expect fun getScreenSize(): Pair<Int, Int>
```

Сколько `actual` реализаций тебе нужно?

- **A)** 1 — общая для всех
- **B)** 2 — для mobile (Android+iOS) и Desktop
- **C)** 3 — по одной для каждого target'а
- **D)** 0 — `getScreenSize` можно получить из Compose

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C** (3 реализации)

Правило: **один `actual` = один target** (или один source set, если несколько target'ов делят source set).

```
androidMain/  → actual fun getScreenSize() = ...  (через WindowManager)
iosMain/      → actual fun getScreenSize() = ...  (через UIScreen)
desktopMain/  → actual fun getScreenSize() = ...  (через java.awt / AWT)
```

> 💡 **Нюанс:** `iosX64`, `iosArm64`, `iosSimulatorArm64` все используют **один** source set `iosMain`. Поэтому для iOS — одна реализация на все три target'а. Но Android и Desktop — разные source set'ы.

Итого: 3 source set'а → 3 `actual`.

</details>

---

## Вопрос 10. Что из этого **НЕ** является причиной использовать KMP вместо Flutter?

- **A)** Хочешь нативный UI (Compose/SwiftUI), а не свой движок рендеринга
- **B)** Уже есть Kotlin-команда, Flutter — новый стек
- **C)** Нужна максимальная производительность на старых устройствах
- **D)** Нужно использовать существующие Java/Kotlin библиотеки без обёрток

- **E)** Все перечисленные — валидные причины

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: E** (все перечисленные — валидные причины)

| Причина | Почему KMP лучше Flutter |
|---------|--------------------------|
| Нативный UI | KMP + Compose Multiplatform = декларативный UI, но рендерится нативно. Flutter рисует сам (Skia) |
| Kotlin-команда | Команда уже знает Kotlin, не нужно учить Dart |
| Производительность | Нативный рендеринг = меньше overhead, чем собственный движок Flutter |
| Java/Kotlin libs | Из commonMain можно напрямую использовать любые JVM-библиотеки (на Android). В Flutter — только через platform channels |

> ⚠️ Это **не** значит, что Flutter плохой. Это значит, что **для разных задач** подходят разные инструменты.

</details>

---

## Вопрос 11. Твой `build.gradle.kts` в shared-модуле:

```kotlin
kotlin {
    androidTarget()
    iosArm64()
    
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
            }
        }
        val androidMain by getting {
            dependencies {
                implementation("com.squareup.okhttp3:okhttp:4.12.0")
            }
        }
    }
}
```

Что будет, если в `commonMain` написать `import okhttp3.OkHttpClient`?

- **A)** Скомпилируется для Android, но не для iOS
- **B)** Скомпилируется для обоих — Gradle подтянет зависимость автоматически
- **C)** Ошибка компиляции: `Unresolved reference: okhttp3`
- **D)** Gradle-ошибка при сборке: «зависимость не найдена в commonMain»

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A** (скомпилируется для Android, не для iOS)

Подожди — это не совсем так! Давай точнее:

**На самом деле:** будет **ошибка компиляции для BOTH** target'ов, потому что `okhttp3` объявлен только в `androidMain`, а `import` стоит в `commonMain`. Компилятор для `commonMain` не видит зависимости из `androidMain`.

Но если бы вопрос был «что будет, если написать `import okhttp3.OkHttpClient` в `androidMain`» — тогда скомпилируется.

> ⚠️ **Ключевой принцип:** source set'ы иерархичны. `commonMain` — **родитель**, `androidMain` — **дочерний**. Дочерний видит всё из родителя, но **не наоборот**. Зависимости из `androidMain` **не видны** в `commonMain`.

**Правильный ответ (точнее): C** — ошибка `Unresolved reference: okhttp3`, потому что в classpath `commonMain` этой зависимости нет.

> 💡 **Правильное решение:** использовать **Ktor Client** вместо OkHttp в `commonMain` (Ktor есть для всех платформ), или объявить `expect`/`actual` для HTTP-клиента.

</details>

---

## Вопрос 12. Финальный: объясни коллеге

Представь, что коллега говорит: *«Зачем KMP? Я могу просто написать на Kotlin для Android и на Swift для iOS. Два разработчика — два проекта. Всё просто».*

Выбери **лучший** ответ:

- **A)** «Потому что JetBrains сказала, что это правильно»
- **B)** «Потому что бизнес-логика (расчёты, валидация, работа с данными) не зависит от платформы. Пишешь один раз → один раз тестируешь → один раз фиксишь баг. А UI — каждый на своём. Это не "заменит" нативную разработку, а убирает дублирование там, где оно бессмысленно»
- **C)** «Потому что Swift — плохой язык»
- **D)** «Не могу объяснить, просто доверься»

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Это и есть суть KMP:

```
┌────────────────────────────────────────────────────────────┐
│  Что KMP убирает (дублирование):                          │
│  • Расчёты, формулы                                       │
│  • Валидация данных                                       │
│  • Работа с БД (SQLDelight)                               │
│  • HTTP-клиент (Ktor)                                     │
│  • Обработка ошибок                                       │
│  • Модели данных (data class)                             │
│  • UI на Compose (если хочешь)                            │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  Что KMP НЕ убирает (остается нативным):                  │
│  • Push (FCM vs APNs)                                     │
│  • Камера, гироскоп                                       │
│  • Специфичные системные API                              │
│  • App Store / Play Store публикации                      │
└────────────────────────────────────────────────────────────┘
```

**Почему не другие:**
- A — «потому что сказали» — не аргумент
- C — ругать чужой стек — это не профессионально и не по теме
- D — «доверься» — это ответ для CEO, не для коллеги-разработчика

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 10–12 | 🎉 Отлично! Можешь смело переходить к Модулю 2 |
| 7–9 | 📖 Перечитай секции 2 и 3, проветри вопрос, где ошибся |
| 4–6 | ⚠️ Пройди модуль ещё раз. Сфокусируйся на expect/actual и source set'ах |
| < 4 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

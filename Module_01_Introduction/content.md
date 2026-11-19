# Модуль 1: Введение в Kotlin Multiplatform и настройка окружения

---

## С чего всё начинается

Представь ситуацию: ты разработчик в продуктовой команде. Пришло ТЗ — сделать приложение для трекинга привычек. Дедлайн через месяц.

И тут выясняется, что нужно **два** приложения:
- Android-версия — на Kotlin + Compose
- iOS-версия — на Swift + SwiftUI

Два разработчика. Два стека. Две базы данных. Два набора багов. И каждый раз, когда меняется бизнес-логика (а она меняется), нужно вносить правку **дважды**. А ещё лучше — дважды с разными ошибками, потому что один разработчик что-то упустил.

Знакомая боль?

**Kotlin Multiplatform (KMP)** — это ответ на эту боль. Одна кодовая база для бизнес-логики, которая компилируется и на Android, и на iOS, и даже на Desktop.

Но давай не будем сразу прыгать в омут. Сначала разберёмся, **как именно** KMP решает эту проблему и **что из кода** можно переиспользовать, а **что** — нет.

---

## Что мы будем делать в этом модуле

| # | Задача | Время |
|---|--------|-------|
| 1 | Понять, как устроен KMP-проект изнутри | ~30 мин |
| 2 | Разобраться с `expect/actual` на пальцах | ~30 мин |
| 3 | Написать первый экран, который работает на двух платформах | ~1 час |
| 4 | Выполнить домашнее задание | ~2 часа |

**Итого:** 5–7 часов. Можно растянуть на два-три вечера.

> 💡 **Важно:** Для iOS-части нужен Mac с Xcode. Если у вас Windows — не паникуйте. Вы всё равно сможете писать общий код, запускать на Android, а iOS-часть просто не запускаете локально. Мы на это учтём.

---

## 1. Архитектура: один код — много платформ

### Аналогия из жизни

Думай о KMP-проекте как о **ресторане с международной франшизой**:

```
                    🏢 Корпоративный офис (commonMain)
                    ┌─────────────────────────────────┐
                    │  Рецепты блюд (бизнес-логика)   │
                    │  Стандарты сервиса (UI-компоненты)│
                    │  Бухгалтерия (модели данных)    │
                    └─────────────────────────────────┘
                         │                    │
            ┌────────────┘                    └────────────┐
            │                                              │
   🇷🇺 Москва                      🇯🇵 Токио
   (androidMain)                  (iosMain)
   ┌──────────────────┐         ┌──────────────────┐
   │ Повар-русский    │         │ Повар-японский   │
   │ готовит по тем   │         │ готовит по тем   │
   │ же рецептам, но  │         │ же рецептам, но  │
   │ использует       │         │ использует       │
   │ свои сковородки  │         │ свои ножи        │
   │ и плиту          │         │ и духовку        │
   └──────────────────┘         └──────────────────┘
```

**Корпоративный офис** (`commonMain`) — это вся логика, которая **не зависит** от платформы: расчёты, бизнес-правила, модели данных, UI-компоненты на Compose.

**Филиалы** (`androidMain`, `iosMain`) — это «склейка» с платформой: запуск активности, доступ к камере, push-уведомления, работа с файловой системой.

### Теперь посмотри на дерево папок

Когда ты создашь KMP-проект, структура будет такой:

```
MyProject/
├── shared/                          ← общий модуль (shared module)
│   └── src/
│       ├── commonMain/kotlin/       ← 🔥 80-90% кода живёт здесь
│       │   ├── model/               ← данные, сущности
│       │   ├── logic/               ← бизнес-логика
│       │   ├── ui/                  ← Compose-экраны
│       │   └── platform/            ← expect-объявления
│       │
│       ├── commonTest/kotlin/       ← тесты общего кода
│       │
│       ├── androidMain/kotlin/      ← 🤖 что-то Android-specific
│       │   └── platform/            ← actual для Android
│       │
│       ├── iosMain/kotlin/          ← 🍎 что-то iOS-specific
│       │   └── platform/            ← actual для iOS
│       │
│       └── desktopMain/kotlin/      ← 💻 (опционально)
│
├── androidApp/                      ← Android-приложение (нативная обёртка)
│   └── src/main/kotlin/
│       └── MainActivity.kt          ← точка входа для Android
│
└── iosApp/                          ← Xcode-проект (нативная обёртка)
    └── iOSApp/
        └── ContentView.swift        ← точка входа для iOS
```

### Ключевой вопрос: что именно можно положить в commonMain?

Вот чек-лист. Отметь, что для тебя очевидно, а что — нет:

| Что | В commonMain? | Почему |
|-----|:---:|--------|
| Модели данных (data class) | ✅ | Чистый Kotlin, не зависит от ОС |
| Бизнес-логика (расчёты, валидация) | ✅ | Чистый Kotlin |
| UI на Compose Multiplatform | ✅ | Компилируется под все платформы |
| Coroutines, Flow | ✅ | Кроссплатформенные |
| SQLDelight (БД) | ✅ | Есть драйверы для всех платформ |
| Ktor Client (сеть) | ✅ | Есть движки для всех платформ |
| `android.util.Log` | ❌ | Это Android API |
| `NSFileManager` | ❌ | Это iOS API |
| Доступ к камере | ❌ | Разные API → через expect/actual |
| Push-уведомления | ❌ | FCM vs APNs → через expect/actual |
| Работа с файловой системой | ❌ | Разные пути → через expect/actual |

> 📌 **Правило:** Если ты используешь только `kotlin.*`, корутины, Compose, SQLDelight, Ktor — ты в `commonMain`. Нужен что-то из ОС? → `expect/actual`.

---

## 2. Expect/Actual: как одна функция живёт на двух платформах

### Проблема

Ты пишешь в `commonMain` функцию «получить имя платформы». На Android ты хочешь получить `"Android"`, на iOS — `"iOS"`. Но API у них разные. Как быть?

### Решение: объявляю контракт, а реализую отдельно

Думай об `expect` как о **заказе в ресторане**: ты говоришь «дайте мне напиток» (объявление), а бармен на каждой станции (платформа) льёт свой (реализация).

```
        commonMain (объявление):
        ┌─────────────────────────────────────────┐
        │  expect fun getPlatformName(): String   │  ← "я знаю, что такая функция есть"
        └─────────────────────────────────────────┘
                   │                        │
                   ▼                        ▼
        androidMain (реализация)    iosMain (реализация)
        ┌─────────────────────────┐  ┌─────────────────────────┐
        │ actual fun              │  │ actual fun              │
        │ getPlatformName():      │  │ getPlatformName():      │
        │ String = "Android"      │  │ String = "iOS"          │
        └─────────────────────────┘  └─────────────────────────┘
```

### Код

Вот как это выглядит. Три файла, три источника:

**Файл 1: `shared/src/commonMain/kotlin/platform/Platform.kt`**

```kotlin
package com.ecotrack.platform

// Объявляем: «такая функция существует, но я не знаю, как она работает»
expect fun getPlatformName(): String
```

**Файл 2: `shared/src/androidMain/kotlin/platform/Platform.kt`**

```kotlin
package com.ecotrack.platform

// Реализация для Android
actual fun getPlatformName(): String = "Android"
```

**Файл 3: `shared/src/iosMain/kotlin/platform/Platform.kt`**

```kotlin
package com.ecotrack.platform

// Реализация для iOS
actual fun getPlatformName(): String = "iOS"
```

**И в любом месте `commonMain` ты просто вызываешь:**

```kotlin
val greeting = "Привет, ${getPlatformName()}!"
// На Android: "Привет, Android!"
// На iOS:     "Привет, iOS!"
```

### Подводные камни

> ⚠️ **Пакеты и имена должны совпадать точно.** `expect` в `commonMain` и `actual` в `androidMain` должны быть в одном и том же пакете, с тем же именем. Иначе компилятор выдаст:
> 
> ```
> e: ACTUAL without corresponding EXPECT
> ```

> ⚠️ **`expect` — это не интерфейс.** Ты не можешь объявить `expect class` с методом и потом реализовать его как хочешь. `expect` — это обещание: «в каждой платформе будет соответствующий `actual`». Для более сложных случаев (разные методы в разных платформах) используй **platform-specific sealed interfaces** или **Service Locator pattern** — разберём в Модуле 7.

### Когда expect/actual НЕ нужен

Частая ошибка новичков — оборачивать в `expect/actual` то, что не зависит от платформы:

```kotlin
// ❌ НЕ надо так
expect fun parseInt(s: String): Int  // это есть в stdlib, работает везде

// ✅ Просто используй
val x = "42".toInt()  // работает на всех платформах из коробки
```

---

## 3. Создаём проект EcoTrack

### Как подойти к решению

Прежде чем нажимать кнопки в IDE, давай разберёмся, **что именно** мы делаем и **зачем** каждый шаг:

1. **Выбираем wizard** — потому что создавать KMP-проект вручную (писать Gradle-скрипты, подключать плагины) — это боль. Wizard делает это за нас.
2. **Выбираем target'ы** — это те платформы, под которые мы компилируем общий код.
3. **Выбираем UI framework** — Compose Multiplatform даст нам декларативный UI на Kotlin.
4. **Смотрим на build.gradle.kts** — чтобы понять, что именно настроено под капотом.

### Пошагово

**Шаг 1: Открой Android Studio (или IntelliJ IDEA)**

Минимальные требования:
- JDK 17+ (встроенная в Android Studio подойдёт)
- Последний плагин **Kotlin Multiplatform** (устанавливается из Plugin Repository)

**Шаг 2: Создай проект**

```
File → New → New Project → Multiplatform Mobile (или Multiplatform Application)
```

**Шаг 3: Настрой параметры**

| Параметр | Значение | Зачем |
|----------|----------|-------|
| Project name | `EcoTrack` | Имя проекта |
| Target | Android, iOS | Компилируем под обе |
| UI Framework | Compose Multiplatform | Declarative UI |
| Kotlin version | последняя стабильная | Не будем джиттерить версии |

Нажми **Finish**. Подожди, пока Gradle скачает зависимости (первый раз может занять 5-10 минут).

**Шаг 4: Изучи `shared/build.gradle.kts`**

Открой файл. Вот ключевые секции:

```kotlin
plugins {
    kotlin("multiplatform")
    kotlin("plugin.compose")          // ← Compose Multiplatform
    kotlin("plugin.serialization")    // ← для JSON (разберём в Модуле 5)
}

kotlin {
    // ─── Целевые платформы ───
    androidTarget {
        // Настройки для Android
    }
    
    iosX64()       // ← симулятор на Intel Mac
    iosArm64()     // ← реальный iPhone (Apple Silicon)
    iosSimulatorArm64() // ← симулятор на Apple Silicon (M1/M2/M3)
    
    // ─── Зависимости, общие для всех платформ ───
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation(compose.runtime)
                implementation(compose.foundation)
                implementation(compose.material3)
                // ...
            }
        }
    }
}
```

> 📌 **Запомни:** всё, что в `commonMain` dependencies — доступно на всех платформах. Если нужна зависимость только для Android — ищи секцию `androidMain`.

<details>
<summary>🔍 Развернуть: полный build.gradle.kts (чтобы сверить с твоим)</summary>

```kotlin
plugins {
    kotlin("multiplatform") version "2.0.20"
    kotlin("plugin.compose") version "2.0.20"
    kotlin("plugin.serialization") version "2.0.20"
    id("com.android.library") version "8.3.0"
}

kotlin {
    androidTarget {
        compilations.all {
            kotlinOptions {
                jvmTarget = "17"
            }
        }
    }
    
    iosX64()
    iosArm64()
    iosSimulatorArm64()
    
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.compose.runtime:runtime:1.6.2")
                implementation("org.jetbrains.compose.foundation:foundation:1.6.2")
                implementation("org.jetbrains.compose.material3:material3:1.6.2")
                implementation("org.jetbrains.compose.ui:ui:1.6.2")
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
                implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")
            }
        }
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
            }
        }
        val androidMain by getting
        val iosMain by getting
    }
}

android {
    namespace = "com.ecotrack.shared"
    compileSdk = 34
    defaultConfig {
        minSdk = 24
    }
}

compose.runtime {
    entryPoints = listOf("androidx.compose.ui.window.Recomposer")
}
```

> ⚠️ Версии могут отличаться в зависимости от того, когда ты создаёшь проект. Не копируй наслепую — смотри, что сгенерировал wizard.

</details>

---

## 4. Первый экран: от идеи до запуска

### Как подойти к решению

Мы хотим написать **один** Compose-экран, который будет:
- Показывать заголовок
- Считать клики по кнопке
- Работать и на Android, и на iOS

Давай разложим на шаги:

```
┌─────────────────────────────────────────────────────────────────────┐
│  commonMain                                                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  MainScreen()                                              │    │
│  │  ┌─────────────────────────────────────────────────────┐    │    │
│  │  │  Text("Добро пожаловать в EcoTrack!")              │    │    │
│  │  │  Text("Привычек добавлено: $count")               │    │    │
│  │  │  Button("Добавить") { count++ }                    │    │    │
│  │  └─────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│              ▲                                    ▲                  │
│              │ import                             │ import           │
└──────────────┼────────────────────────────────────┼──────────────────┘
               │                                    │
     ┌─────────┴──────────┐               ┌────────┴────────────┐
     │  androidMain       │               │  iosMain            │
     │  MainActivity      │               │  MainViewController │
     │  setContent {       │              │  ComposeUIVC {      │
     │    MainScreen()     │              │    MainScreen()     │
     │  }                  │              │  }                  │
     └─────────────────────┘              └─────────────────────┘
```

Видишь? **Один и тот же** `MainScreen()` вставляется в две разные «обёртки». Android-обёртка — это `ComponentActivity` с `setContent`. iOS-обёртка — это `ComposeUIViewController`. А сам экран — один, общий.

### Пишем код

**Шаг 1: Общий экран**

Создай файл `shared/src/commonMain/kotlin/ui/MainScreen.kt`:

<details>
<summary>🔍 Развернуть: код MainScreen.kt</summary>

```kotlin
package com.ecotrack.ui

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.height
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableIntStateOf
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import com.ecotrack.platform.getPlatformName

@Composable
fun MainScreen() {
    var count by mutableIntStateOf(0)

    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            text = "Добро пожаловать в EcoTrack!",
            style = MaterialTheme.typography.headlineMedium
        )

        Spacer(modifier = Modifier.height(16.dp))

        Text(
            text = getPlatformName(),
            style = MaterialTheme.typography.bodyMedium
        )

        Spacer(modifier = Modifier.height(16.dp))

        Text(text = "Привычек добавлено: $count")

        Spacer(modifier = Modifier.height(24.dp))

        Button(onClick = { count++ }) {
            Text("Добавить привычку")
        }
    }
}
```

> 💡 **Примечание:** Здесь мы уже используем `getPlatformName()` из модуля `platform` — тот самый `expect/actual`, который мы разбирали выше. Так что этот экран уже демонстрирует кроссплатформенность: на Android будет написано "Android", на iOS — "iOS".

</details>

**Шаг 2: Android-обёртка**

Открой `androidApp/src/main/kotlin/MainActivity.kt`:

<details>
<summary>🔍 Развернуть: код MainActivity.kt (Android)</summary>

```kotlin
package com.ecotrack

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.ui.Modifier
import com.ecotrack.ui.MainScreen

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    MainScreen()
                }
            }
        }
    }
}
```

> 📌 **Что здесь происходит:** `ComponentActivity` — это стандартная Android-активность. Метод `setContent { }` — это мост между Activity и Compose. Внутри мы вызываем **тот же самый** `MainScreen()` из `commonMain`. Всё.

</details>

**Шаг 3: iOS-обёртка**

В iOS-части Xcode-проекта есть Swift-файл, который создаёт `UIViewController`. Но с Compose Multiplatform мы делаем это проще — через Kotlin:

Создай `shared/src/iosMain/kotlin/MainViewController.kt`:

<details>
<summary>🔍 Развернуть: код MainViewController.kt (iOS)</summary>

```kotlin
package com.ecotrack

import androidx.compose.ui.window.ComposeUIViewController
import com.ecotrack.ui.MainScreen

fun MainViewController() = ComposeUIViewController {
    MainScreen()
}
```

Вот и всё. Одна строка. `ComposeUIViewController` — это обёртка, которая создаёт `UIViewController`, внутри которого рендерится наш Compose-дерево.

</details>

Затем в Swift-коде Xcode-проекта (`iosApp/iOSApp/ContentView.swift`):

```swift
import SwiftUI
import shared

struct ContentView: View {
    var body: some View {
        MainViewController()
            .ignoresSafeArea()
    }
}
```

> 📌 `import shared` — это импорт Kotlin-модуля в Swift. KMP экспортирует Swift-фреймворк из commonMain/iosMain, и Xcode видит его как обычный Swift-модуль.

**Шаг 4: Запуск**

| Платформа | Как запустить |
|-----------|---------------|
| Android | Выбери эмулятор в run-конфигурации → ▶️ |
| iOS | Выбери iOS-симулятор → ▶️ (нужен Mac) |

**Ожидаемый результат:**

```
┌─────────────────────────────┐
│                             │
│   Добро пожаловать в        │
│       EcoTrack!             │
│                             │
│         Android             │  ← на Android
│         iOS                 │  ← на iOS
│                             │
│   Привычек добавлено: 0     │
│                             │
│  ┌───────────────────────┐  │
│  │  Добавить привычку    │  │
│  └───────────────────────┘  │
│                             │
└─────────────────────────────┘
```

Нажал кнопку 3 раза → «Привычек добавлено: 3». Одинаково на обеих платформах.

---

## 5. Подводные камни первого модуля

Прежде чем перейти к домашнему заданию, давай разберём 3 частые ошибки:

### ❌ Ошибка 1: Путаница с папками

```
// ❌ Так делать НЕ надо
shared/src/commonMain/kotlin/androidMain/...  ← двойная вложенность, будет конфликтовать
```

**Правильно:** `commonMain` и `androidMain` — **сестринские** папки внутри `src/`, а не вложенные друг в друга.

```
src/
├── commonMain/    ← уровень 1
├── androidMain/   ← уровень 1 (сестра commonMain)
└── iosMain/       ← уровень 1 (сестра commonMain)
```

### ❌ Ошибка 2: Забыл добавить actual для всех платформ

Ты объявил `expect` и написал `actual` для Android, но забыл для iOS. Компилятор для iOS упадёт:

```
e: Expected an actual declaration for 'getPlatformName(): String'
```

**Решение:** При объявлении `expect` — сразу создавай файлы `actual` для **всех** target'ов.

### ❌ Ошибка 3: Неправильная версия Compose

```kotlin
// ❌ Версии Kotlin и Compose не совпадают
kotlin("multiplatform") version "2.0.20"
kotlin("plugin.compose") version "1.9.22"  // ← несовместимо!
```

**Решение:** Wizard подставит совместимые версии. Если меняешь одну — меняй обе.

---

## 6. Домашнее задание

> 🎯 **Цель:** убедить себя (и компилятор), что ты понял разницу между `commonMain` и платформенными source set'ами.

---

### Задание 1: Измени UI

**Что сделать:** Сделай кнопку зелёной.

**Как думать:** В Material 3 цвет кнопки задаётся через `ButtonDefaults.buttonColors()`. Нам нужно передать `containerColor`. Импортируй `Color` из `androidx.compose.ui.graphics`.

<details>
<summary>✅ Развернуть: решение</summary>

```kotlin
import androidx.compose.material3.ButtonDefaults
import androidx.compose.ui.graphics.Color

Button(
    onClick = { count++ },
    colors = ButtonDefaults.buttonColors(
        containerColor = Color(0xFF4CAF50)
    )
) {
    Text("Добавить привычку")
}
```

</details>

---

### Задание 2: Expect/Actual с приветствием

**Что сделать:** Создай свой `expect/actual` — функцию `getGreeting(): String`.

**Как думать:**
1. В `commonMain/platform/` создай файл `Greeting.kt` — напиши `expect fun getGreeting(): String`
2. В `androidMain/platform/` — `actual fun getGreeting() = "Привет, Android!"`
3. В `iosMain/platform/` — `actual fun getGreeting() = "Привет, iOS!"`
4. В `MainScreen` добавь `Text(getGreeting())` под заголовком

Подсказка: пакеты должны совпадать с `expect` (тот же `package com.ecotrack.platform`).

<details>
<summary>✅ Развернуть: решение</summary>

**`commonMain/platform/Greeting.kt`:**
```kotlin
package com.ecotrack.platform

expect fun getGreeting(): String
```

**`androidMain/platform/Greeting.kt`:**
```kotlin
package com.ecotrack.platform

actual fun getGreeting(): String = "Привет, Android!"
```

**`iosMain/platform/Greeting.kt`:**
```kotlin
package com.ecotrack.platform

actual fun getGreeting(): String = "Привет, iOS!"
```

**В `MainScreen`:**
```kotlin
// Добавь после заголовка:
Spacer(modifier = Modifier.height(8.dp))
Text(
    text = getGreeting(),
    style = MaterialTheme.typography.titleMedium
)
```

</details>

---

### Задание 3: Тёмная тема

**Что сделать:** Добавь переключатель тёмной/светлой темы.

**Как думать:**
- Нужна переменная состояния: `var isDarkMode by mutableStateOf(false)`
- Кнопка (или переключатель) меняет её
- Фон экрана должен меняться: `Surface(color = if (isDarkMode) Color(0xFF1A1A2E) else Color.White)`

Подсказка: оберни всё содержимое в `Surface` с динамическим `color`.

<details>
<summary>✅ Развернуть: решение</summary>

```kotlin
@Composable
fun MainScreen() {
    var count by mutableIntStateOf(0)
    var isDarkMode by mutableStateOf(false)

    val backgroundColor = if (isDarkMode) Color(0xFF1A1A2E) else Color.White
    val textColor = if (isDarkMode) Color.White else Color.Black

    Surface(
        modifier = Modifier.fillMaxSize(),
        color = backgroundColor
    ) {
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(24.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            Text(
                text = "Добро пожаловать в EcoTrack!",
                style = MaterialTheme.typography.headlineMedium,
                color = textColor
            )

            Spacer(modifier = Modifier.height(16.dp))

            Text(
                text = getPlatformName(),
                style = MaterialTheme.typography.bodyMedium,
                color = textColor
            )

            Spacer(modifier = Modifier.height(16.dp))

            Text(
                text = "Привычек добавлено: $count",
                color = textColor
            )

            Spacer(modifier = Modifier.height(16.dp))

            Text(
                text = getGreeting(),
                style = MaterialTheme.typography.titleMedium,
                color = textColor
            )

            Spacer(modifier = Modifier.height(24.dp))

            Button(
                onClick = { count++ },
                colors = ButtonDefaults.buttonColors(containerColor = Color(0xFF4CAF50))
            ) {
                Text("Добавить привычку")
            }

            Spacer(modifier = Modifier.height(12.dp))

            Button(
                onClick = { isDarkMode = !isDarkMode },
                colors = ButtonDefaults.buttonColors(
                    containerColor = if (isDarkMode) Color(0xFFFFC107) else Color(0xFF2196F3)
                )
            ) {
                Text(if (isDarkMode) "☀️ Светлая тема" else "🌙 Тёмная тема")
            }
        }
    }
}
```

</details>

---

### Задание 4 (бонус): Счётчик с анимацией

**Что сделать:** При нажатии кнопки цифра «перескакивает» с fade-эффектом.

**Как думать:**
- Используй `AnimatedContent` из `androidx.compose.animation`
- Ключ анимации — значение `count`
- Вход: `fadeIn + slideIn`
- Выход: `fadeOut + slideOut`

<details>
<summary>✅ Развернуть: решение</summary>

```kotlin
import androidx.compose.animation.AnimatedContent
import androidx.compose.animation.fadeIn
import androidx.compose.animation.fadeOut
import androidx.compose.animation.slideInHorizontally
import androidx.compose.animation.slideOutHorizontally
import androidx.compose.foundation.layout.size
import androidx.compose.ui.text.style.TextAlign

AnimatedContent(
    targetState = count,
    transitionSpec = {
        (fadeIn(animationSpec = tween(300)) + slideInHorizontally { it / 2 }) to
        (fadeOut(animationSpec = tween(300)) + slideOutHorizontally { -it / 2 })
    }
) { currentCount ->
    Text(
        text = "Привычек добавлено: $currentCount",
        style = MaterialTheme.typography.titleLarge,
        textAlign = TextAlign.Center
    )
}
```

> ⚠️ `AnimatedContent` — из `androidx.compose.animation`. В `build.gradle.kts` commonMain добавь:
> ```kotlin
> implementation("org.jetbrains.compose.animation:animation:1.6.2")
> ```

</details>

---

### Критерии сдачи

| Задание | Критерий | Статус |
|---------|----------|:------:|
| 1 | Кнопка зелёная на обеих платформах | ☐ |
| 2 | При запуске на Android видно "Привет, Android!", на iOS — "Привет, iOS!" | ☐ |
| 3 | Кнопка переключает фон экрана, текст остаётся читаемым | ☐ |
| 4 | (бонус) Цифра анимируется при изменении | ☐ |

---

## Чек-лист перед переходом к Модулю 2

- [ ] Понимаю, **зачем** нужен `commonMain` и **что** в него нельзя класть
- [ ] Могу объяснить коллегам, как работает `expect/actual`, без заиканий
- [ ] Мой проект запускается на Android (и на iOS, если есть Mac)
- [ ] DOM задание сдано (минимум задания 1–3)
- [ ] Я не боюсь открывать `build.gradle.kts` и что-то в нём менять

---

> 📝 **Самопроверка:** [Квиз к Модулю 1](./quiz.md) — 12 вопросов для проверки понимания. Пройди перед переходом к следующему модулю.
>
> 🔜 **Следующий модуль:** Основы Compose Multiplatform. Будет про layout-систему, состояние и первые компоненты Material 3. Если этот модуль для тебя был «лёгким прогоном» — отлично, в следующих будет жарко.

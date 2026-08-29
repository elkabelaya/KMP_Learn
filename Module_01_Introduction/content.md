# 📘 Модуль 1: Введение в Kotlin Multiplatform и настройка окружения

**Добро пожаловать на курс!**  
В этом модуле мы заложим фундамент. Вы поймете, как работает магия KMP (Kotlin Multiplatform), почему это не просто «еще один фреймворк», а философия разработки, и создадим первый набросок нашего приложения **EcoTrack**.

**Цели модуля:**
1. Понять архитектуру KMP-проекта (общий код vs платформенный)
2. Настроить окружение для разработки под Android и iOS
3. Освоить механизм `expect/actual`
4. Запустить первый экран на Compose Multiplatform

**Время выполнения:** ~5–7 часов.

---

## 1. Введение: Зачем нам KMP?

Раньше, чтобы сделать приложение на Android и iOS, нужно было писать два разных проекта:
- **Android:** Kotlin + Jetpack Compose
- **iOS:** Swift + SwiftUI (или UIKit)

Это означало дублирование бизнес-логики, баги в одной версии не фиксировались в другой и двойная работа по поддержке.

**Kotlin Multiplatform (KMP)** позволяет написать бизнес-логику **один раз** на Kotlin и использовать её везде.
- **UI (Интерфейс):** Раньше это было сложно, но с появлением **Compose Multiplatform** мы можем писать UI тоже на Kotlin.
- **Бизнес-логика:** Базы данных, сети, валидация — всё это пишется один раз.

**Наш проект «EcoTrack»:**
Мы будем создавать трекер эко-привычек. Пользователь нажимает кнопку «Сдал пластик», и это действие сохраняется в базе данных, отправляется на сервер и отображается в статистике. Всё это будет работать одинаково на Android, iOS и даже Desktop.

---

## 2. Теория: Структура проекта KMP

В обычном Android-проекте у вас есть папка `src/main`. В KMP структура сложнее, потому что код делится на **общий** и **платформенный**.

### 📂 Дерево папок (упрощенно)
```text
src/
├── commonMain/          <-- ОБЩИЙ КОД (Shared)
│   └── kotlin/          <-- Логика, которая работает везде
├── commonTest/          <-- Тесты для общего кода
├── androidMain/         <-- КОД ТОЛЬКО ДЛЯ ANDROID
│   └── kotlin/          <-- MainActivity, Android-специфики
├── androidUnitTest/     <-- Тесты для Android
├── iosMain/            <-- КОД ТОЛЬКО ДЛЯ iOS (Swift/Kotlin)
│   └── kotlin/          <-- AppDelegate, iOS-специфики
├── iosXCTest/           <-- Тесты для iOS
└── ... (другие платформы)
```

### 🔑 Ключевое правило:
- **`commonMain`**: Здесь живет 80–90% вашего кода (бизнес-логика, модели данных, UI на Compose). Этот код компилируется и для Android, и для iOS.
- **`androidMain` / `iosMain`**: Здесь живет код, который зависит от конкретной ОС (например, доступ к камере, уведомления, запуск приложения).

---

## 3. Теория: Механизм Expect/Actual

Это самая важная концепция KMP. Как сделать так, чтобы в `commonMain` мы могли вызвать функцию «Открыть камеру», хотя на Android и iOS API для этого разные?

Мы используем механизм **Expect/Actual**.

1. В `commonMain` мы объявляем функцию как **`expect`** (ожидаем, что она будет реализована где-то еще).
2. В `androidMain` и `iosMain` мы пишем **`actual`** (реальную реализацию) для каждой платформы.

### Пример кода:
```kotlin
// 📁 src/commonMain/kotlin/com.ecotrack/platform/Platform.kt
package com.ecotrack.platform

// Объявляем, что функция существует, но не пишем её тело
expect fun getPlatformName(): String

// 📁 src/androidMain/kotlin/com.ecotrack/platform/Platform.kt
package com.ecotrack.platform

// Реализация для Android
actual fun getPlatformName(): String = "Android"

// 📁 src/iosMain/kotlin/com.ecotrack/platform/Platform.kt
package com.ecotrack.platform

// Реализация для iOS
actual fun getPlatformName(): String = "iOS"
```

**В `commonMain` мы просто пишем:**
```kotlin
println("Привет, ${getPlatformName()}!") 
// На Android выведет "Android", на iOS — "iOS"
```

---

## 4. Практика: Создание проекта EcoTrack

### Шаг 1: Подготовка окружения
- **Android:** Установите Android Studio (с последним плагином Kotlin).
- **iOS:** Для разработки под iOS **обязательно нужен Mac** с установленным Xcode. Если у вас Windows, вы сможете писать код и запускать на Android, но iOS-часть придется тестировать через облачные сервисы (например, Bitrise) или попросить коллегу с Mac.
- **JDK:** Убедитесь, что у вас стоит JDK 17 или выше.

### Шаг 2: Создание проекта
Мы используем официальный **Kotlin Multiplatform Wizard** (встроен в IntelliJ IDEA / Android Studio).

1. Откройте IDE -> **New Project**.
2. Выберите шаблон: **Kotlin Multiplatform Mobile** (или просто **Multiplatform Application**).
3. В настройках выберите:
   - **Target:** Android, iOS (если есть Mac).
   - **UI Framework:** Compose Multiplatform.
4. Назовите проект: `EcoTrack`.
5. Нажмите **Finish**.

### Шаг 3: Изучение `build.gradle.kts`
Откройте файл `shared/build.gradle.kts`. Вы увидите зависимости. Обратите внимание на секцию `kotlin { ... }`:
```kotlin
androidTarget() // Настройка для Android
iosX64()        // Симулятор iOS (Intel)
iosArm64()      // Реальный iPhone (Apple Silicon)
```

---

## 5. Практика: Первый экран на Compose Multiplatform

Теперь напишем код, который будет работать и на Android, и на iOS.

### Шаг 1: Создаем общий UI
Откройте файл `src/commonMain/kotlin/com/ecotrack/ui/MainScreen.kt` (или создайте его, если шаблон не создал).

Мы сделаем экран приветствия с кнопкой.

```kotlin
package com.ecotrack.ui

import androidx.compose.foundation.layout.*
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier

// Это функция, которая будет работать на ВСЕХ платформах
@Composable
fun MainScreen() {
    var count by remember { mutableStateOf(0) }

    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            text = "Добро пожаловать в EcoTrack!",
            style = MaterialTheme.typography.headlineMedium
        )

        Spacer(modifier = Modifier.height(24.dp))

        Text(text = "Привычек добавлено: $count")

        Spacer(modifier = Modifier.height(16.dp))

        Button(onClick = { count++ }) {
            Text("Добавить привычку")
        }
    }
}
```

### Шаг 2: Подключаем UI к платформам (Android)
Откройте `src/androidMain/kotlin/com/ecotrack/MainActivity.kt`.

```kotlin
package com.ecotrack

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.ui.Modifier
// Импортируем наш общий экран
import com.ecotrack.ui.MainScreen

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Запускаем Compose UI
        setContent {
            MaterialTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    // Вызываем общий экран
                    MainScreen()
                }
            }
        }
    }
}
```

### Шаг 3: Подключаем UI к платформам (iOS)
Откройте `src/iosMain/kotlin/com/ecotrack/MainViewController.kt` (или `AppDelegate`, в зависимости от шаблона).

```kotlin
package com.ecotrack

import androidx.compose.ui.window.ComposeUIViewController
// Импортируем наш общий экран
import com.ecotrack.ui.MainScreen

fun MainViewController() = ComposeUIViewController { 
    // Вызываем тот же самый общий экран
    MainScreen() 
}
```

### Шаг 4: Запуск
1. **Android:** Выберите эмулятор в строке запуска и нажмите кнопку Play (зеленый треугольник).
2. **iOS:** Если у вас Mac, выберите симулятор (iPhone 15) и нажмите Play.

**Результат:** Вы должны увидеть экран с заголовком «Добро пожаловать в EcoTrack!» и кнопкой. При нажатии счетчик должен увеличиваться.

---

## 📝 Домашнее задание (Модуль 1)

Чтобы закрепить материал, выполните следующие шаги в своем проекте **EcoTrack**:

1. **Настройка:** Убедитесь, что проект собирается и запускается на Android-эмуляторе. (Если нет Mac — пропустите запуск iOS, но проверьте код).

2. **Изменение UI:** В файле `MainScreen.kt` измените цвет кнопки на зеленый (используйте `ButtonDefaults.buttonColors(containerColor = Color(0xFF4CAF50))`).

3. **Ожидание/Реализация:** Создайте свой пример `expect/actual`.
   - В `commonMain` создайте файл `Platform.kt` с функцией `expect fun getGreeting(): String`.
   - В `androidMain` реализуйте её: возвращает "Привет, Android!".
   - В `iosMain` реализуйте её: возвращает "Привет, iOS!".
   - Выведите результат этой функции на экран (добавьте `Text(text = getGreeting())` в `MainScreen`).

4. **Состояние:** Добавьте переменную `var isDarkMode by remember { mutableStateOf(false) }`. Сделайте кнопку, которая меняет эту переменную. Если `true` — фон экрана должен стать темным (используйте `Surface(color = if(isDarkMode) Color.Black else Color.White)`).

**Критерий сдачи:**
- Приложение запускается.
- Кнопка меняет счетчик.
- Выводится текст, зависящий от платформы (Android/iOS).

---

**💡 Совет:** Не пытайтесь запомнить весь синтаксис Gradle или Compose сразу. Главное сейчас — понять, **где** находится код и **как** он запускается на разных устройствах. В следующих модулях мы будем углубляться в детали.

Удачи! Если возникнут ошибки при сборке — проверьте версии Kotlin и Compose в `build.gradle.kts`, они должны совпадать.

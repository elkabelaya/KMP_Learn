# 📘 Модуль 2: UI-разработка на Compose Multiplatform

**Добро пожаловать во второй модуль!**  
В первом модуле мы запустили «Hello World». Теперь пришло время превратить это в настоящее приложение. Мы научимся создавать списки, переходить между экранами, настраивать темы и правильно работать с системными панелями.

**Цели модуля:**
1. Освоить основные компоновки Compose (Column, Row, Box) и модификаторы
2. Научиться создавать эффективные списки (`LazyColumn`)
3. Реализовать навигацию между экранами (Navigation Compose)
4. Настроить тему приложения (Material3) и переключение цветов
5. **Освоить работу с Scaffold и отступами от системных панелей (Safe Areas)**

**Время выполнения:** ~10–12 часов.

---

## 1. Теория: Layout-система и Модификаторы

Compose — это декларативный UI. Вы описываете *что* хотите увидеть, а фреймворк рисует это.

### Базовые контейнеры
- **`Column`**: Вертикальный список элементов (один под другим)
- **`Row`**: Горизонтальный список (один за другим)
- **`Box`**: Элементы накладываются друг на друга (полезно для иконок поверх текста)

### Модификаторы (`Modifier`)
Это «настройки» для компонентов. Они всегда идут первым аргументом.

- `Modifier.padding(16.dp)` — отступы
- `Modifier.fillMaxWidth()` — растянуть на всю ширину
- `Modifier.clickable { ... }` — сделать кликабельным

**Важно:** Модификаторы можно цеплять через точку:
```kotlin
Button(
    modifier = Modifier.padding(16.dp).fillMaxWidth(), 
    onClick = { ... }
)
```

### Списки: `LazyColumn` vs `Column`
Если у вас 10 элементов — используйте `Column`. Если 100 или бесконечный список (как лента новостей) — **обязательно** используйте `LazyColumn`. Он рендерит только те элементы, которые видны на экране.

---

## 2. Теория: Навигация (Navigation)

В KMP мы используем type-safe навигацию через библиотеку **`androidx.navigation.compose`**. Она работает одинаково на Android и iOS.

**Type-safe навигация (типобезопасная навигация)** в Kotlin Multiplatform (KMP) — это подход к перемещению между экранами приложения, при котором все аргументы и маршруты проверяются на этапе компиляции, а не во время выполнения приложения. Этот механизм стал официально доступен в Jetpack Compose Navigation (начиная с версии 2.8.0), которая сейчас активно используется и развивается в экосистеме KMP (Compose Multiplatform).
**Как это было раньше**: маршруты в Compose Navigation задавались в виде обычных строк (String), похожих на веб-ссылки:"profile/{userId}/{username}". Минусы такого подхода: Ошибки в опечатках, ручной парсинг из NavBackStackEntry и приведение к нужным типам, сложно передавать объекты (приходилось сериализовать в JSON строку).
**Как это работает теперь**: каждый экран и его аргументы описываются с помощью Kotlin-классов или объектов, помеченных аннотацией @Serializable из библиотеки kotlinx.serialization
```kotlin
import androidx.navigation.NavArgument
import kotlinx.serialization.Serializable

@Serializable  
data class HabitDetailArgs(val habitId: String)

composable<HabitDetailArgs> { args ->  
    HabitDetailScreen(habitId = args.habitId)
}

// Навигация:
navController.navigate(HabitDetailArgs(habitId = "123"))  
```

### 🔗 Подключение библиотеки навигации

**⚠️** В Kotlin Multiplatform проектах зависимости должны добавляться **в соответствующих sourceSets** внутри блока `kotlin { ... }`, а не в обычный блок `dependencies`.

#### Вариант 1: Gradle Kotlin DSL (`.gradle.kts`) — Рекомендуемый

Откройте файл `shared/build.gradle.kts` и добавьте зависимости **внутри блока kotlin**:

```
kotlin {
    sourceSets {
        commonMain.dependencies {
            // ✅ Добавляем навигацию в commonMain (общий код)
            implementation("androidx.navigation:navigation-compose:2.7.6")
        }
        
        androidMain.dependencies {
            // Здесь можно добавить Android-специфичные зависимости
        }
        
        iosMain.dependencies {
            // Здесь можно добавить iOS-специфичные зависимости
        }
    }
}
```
или в случаях когда невозможно подключить через sourceSets
```kotlin
dependencies {
    // ✅ ПРАВИЛЬНО для KMP (внутри kotlin { ... })
    commonMainImplementation("androidx.navigation:navigation-compose:2.7.6")
}
```

**Почему это важно:**
- `commonMain.dependencies/commonMainImplementation` — зависимость будет доступна в общем коде (работает на всех платформах)
- `androidMain.dependencie/androidMainImplementation` — только для Android
- `iosMain.dependencies/iosMainImplementation` — только для iOS

#### Вариант 2: Gradle Version Catalog (`libs.versions.toml`) — Современный подход

Если вы используете **Version Catalog** (рекомендуется для новых проектов), создайте или отредактируйте файл `gradle/libs.versions.toml` в корне проекта:

```toml
[versions]
navigation-compose = "2.7.6"
serialization = "1.6.2"

[libraries]
navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigation-compose" }
serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "serialization" }
```

Затем в `shared/build.gradle.kts` используйте алиас **внутри sourceSets**:

```kotlin
kotlin {
    sourceSets {
        commonMain.dependencies {
            // ✅ Используем алиас из Version Catalog
            implementation(libs.navigation.compose)
            implementation(libs.serialization.json)
        }
    }
}
```

**Преимущества Version Catalog:**
- Централизованное управление версиями зависимостей
- Легко обновлять версии в одном месте
- Избегает дублирования версий

### 📦 Подключение плагина сериализации (ВАЖНО для type-safe навигации!)

#### Шаг 1: Добавьте плагин в `settings.gradle.kts` (корень проекта)
***Объявление версии глобально в settings.gradle.kts***
```
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
    plugins {
        // Фиксируем версию плагина для всего проекта
        id("org.jetbrains.kotlin.plugin.serialization") version "2.0.21" 
    }
}
```

***или Через Version Catalogs (Рекомендуемый)***
```
# toml

[versions]
kotlin = "2.0.21" # ваша версия Kotlin

[plugins]
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```
```
# build.gradle.kts
plugins {
    alias(libs.plugins.kotlin.serialization)
}
```

#### Шаг 2: Добавьте зависимость сериализации в `shared/build.gradle.kts`
```
kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.2")
        }
    }
}
```

### Основные понятия навигации:
1. **`NavHost`**: Главный контейнер, который управляет экранами
2. **`NavController`**: Объект, который переключает экраны (`navController.navigate("addHabit")`)
3. **`@Serializable`**: Аннотация для type-safe навигации (NavArg) 

**Проблема KMP:**
На iOS кнопка «Назад» в навигации работает нативно, но нам нужно убедиться, что Compose-навигация корректно обрабатывает системную кнопку «Back». В CMP это работает из коробки, но важно правильно настроить `NavHost`.

---

## 3. Теория: Темизация (Material3)

Мы будем использовать **Material Design 3**. Это позволяет легко переключать темы.

**Цветовая схема (`ColorScheme`):**
В `commonMain` мы определяем цвета. В платформенных модулях (`androidMain`, `iosMain`) мы говорим системе, когда применять темную тему.

```kotlin
// commonMain/kotlin/ui/theme/Theme.kt
@Composable
fun EcoTrackTheme(
    darkTheme: Boolean = false, // Можно управлять переменной
    content: @Composable () -> Unit
) {
    val colors = if (darkTheme) DarkColorScheme else LightColorScheme
    
    MaterialTheme(
        colorScheme = colors,
        content = content
    )
}
```

---

## 4. Теория: Scaffold и Safe Areas (Отступы от системных панелей)

### 🏗️ Что такое Scaffold?

**Scaffold** — это базовый компонент Material3, который обеспечивает стандартную структуру экрана приложения. Он автоматически управляет отступами и размещением основных элементов интерфейса.

**Основные компоненты Scaffold:**
- **`topBar`** — Верхняя панель (AppBar, Toolbar)
- **`bottomBar`** — Нижняя панель (Bottom Navigation Bar)
- **`floatingActionButton`** — Плавающая кнопка действия (FAB)
- **`snackbarHost`** — Хост для уведомлений (Snackbar)
- **`content`** — Основной контент экрана

### 📱 Проблема Safe Areas (Безопасные зоны)

На разных устройствах есть системные панели, которые могут перекрывать контент:

**Android:**
- Статус-бар сверху (время, батарея)
- Навигационные кнопки снизу (или жестовая область)

**iOS:**
- **Notch** (вырез под камеру) — iPhone X и новее
- **Dynamic Island** — iPhone 14 Pro и новее
- **Home Indicator** (полоска снизу) — iPhone без кнопки Home

Если не учесть эти области, ваш контент может быть **недоступен** или **перекрываться**!

### ✅ Решение: WindowInsets и SystemBars

В Compose Multiplatform мы используем **`WindowInsets`** для управления отступами.

#### Вариант 1: Автоматические отступы (Рекомендуется)

Scaffold автоматически добавляет отступы от системных панелей:

```kotlin
@Composable
fun HomeScreen() {
    Scaffold(
        // topBar автоматически учитывает статус-бар и notch
        topBar = { TopAppBar(title = { Text("Привычки") }) },
        
        // bottomBar автоматически учитывает home indicator на iOS
        bottomBar = { BottomNavigation() },
        
        // content автоматически получает отступы
        content = { paddingValues ->
            LazyColumn(
                modifier = Modifier.fillMaxSize(),
                contentPadding = paddingValues // ✅ ВАЖНО!
            ) {
                // Ваш контент
            }
        }
    )
}
```

#### Вариант 2: Ручное управление отступами (Продвинутое)

Если вам нужен полный контроль, используйте `WindowInsets`:

```kotlin
import androidx.compose.foundation.layout.WindowInsets
import androidx.compose.foundation.layout.statusBars
import androidx.compose.foundation.layout.navigationBars

@Composable
fun HomeScreen() {
    Scaffold(
        content = { paddingValues ->
            Column(
                modifier = Modifier
                    .fillMaxSize()
                    .padding(paddingValues) // Применяем отступы от Scaffold
            ) {
                Text("Контент с правильными отступами")
            }
        }
    )
}
```

### 🚨 Частые ошибки при работе с отступами:

**❌ ОШИБКА 1:** Не использовать `contentPadding` в LazyColumn
```kotlin
// Контент будет скрыт под системными панелями!
LazyColumn(
    modifier = Modifier.fillMaxSize() // ❌ Нет отступов!
) { ... }
```

**✅ ПРАВИЛЬНО:** Передавать `paddingValues` из Scaffold
```kotlin
Scaffold(content = { paddingValues ->
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = paddingValues // ✅ Отступы от системных панелей
    ) { ... }
})
```

**❌ ОШИБКА 2:** Дублирование отступов
```kotlin
// Не делайте так! Отступы будут добавлены дважды.
Scaffold(
    content = { paddingValues ->
        Column(
            modifier = Modifier.padding(16.dp) // ❌ Дублирование!
        ) { ... }
    }
)
```

**✅ ПРАВИЛЬНО:** Использовать только `paddingValues` или добавлять свои отступы внутри
```kotlin
Scaffold(
    content = { paddingValues ->
        Column(
            modifier = Modifier.padding(paddingValues) // ✅ Только отступы Scaffold
        ) { ... }
    }
)
```

### 📋 Когда использовать Scaffold?

**Используйте Scaffold, когда:**
- ✅ У вас есть AppBar (верхняя панель)
- ✅ У вас есть Bottom Navigation или Bottom Bar
- ✅ У вас есть FAB (плавающая кнопка)
- ✅ Вы хотите автоматически обрабатывать отступы на iOS и Android

**Не используйте Scaffold, когда:**
- ❌ У вас полноэкранный контент (фото, видео)
- ❌ Вы создаете кастомный layout без стандартных элементов

---

## 5. Практика: Реализация списка привычек

Согласно дизайну из Figma, нам нужен список карточек.

### Шаг 1: Создаем модель данных
В `commonMain` создайте файл `Habit.kt`.

```kotlin
package com.ecotrack.data.model

data class Habit(
    val id: String,
    val title: String,
    val category: String, // "Транспорт", "Мусор"
    val isCompleted: Boolean = false,
    val date: String // Формат "24.10"
)
```

### Шаг 2: Создаем компонент карточки (`HabitCard`)
Создайте файл `commonMain/kotlin/ui/components/HabitCard.kt`.

```kotlin
package com.ecotrack.ui.components

import androidx.compose.foundation.layout.*
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight

@Composable
fun HabitCard(
    habit: com.ecotrack.data.model.Habit,
    onClick: () -> Unit // Клик по карточке
) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp, vertical = 8.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp),
        onClick = onClick
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            // Иконка (заглушка)
            Icon(
                imageVector = androidx.compose.material.icons.Icons.Default.Check, 
                contentDescription = null,
                tint = if (habit.isCompleted) Color(0xFF4CAF50) else MaterialTheme.colorScheme.onSurfaceVariant,
                modifier = Modifier.size(32.dp)
            )

            Spacer(modifier = Modifier.width(16.dp))

            Column {
                Text(
                    text = habit.title,
                    style = MaterialTheme.typography.titleMedium,
                    fontWeight = FontWeight.Bold
                )
                Text(
                    text = habit.category,
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }

            Spacer(modifier = Modifier.weight(1f)) // Толкает дату вправо

            Text(
                text = habit.date,
                style = MaterialTheme.typography.bodySmall,
                color = Color.Gray
            )
        }
    }
}
```

### Шаг 3: Экран списка (`HomeScreen`)
Создайте файл `commonMain/kotlin/ui/screens/HomeScreen.kt`.

```kotlin
package com.ecotrack.ui.screens

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier

@Composable
fun HomeScreen(
    habits: List<com.ecotrack.data.model.Habit>, // Данные из ViewModel (пока заглушка)
    onAddClick: () -> Unit,
    onHabitClick: (String) -> Unit // ID привычки
) {
    Scaffold(
        // Верхняя панель (AppBar)
        topBar = {
            TopAppBar(
                title = { Text("Мои привычки") },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer,
                    titleContentColor = MaterialTheme.colorScheme.onPrimaryContainer
                )
            )
        },
        
        // Плавающая кнопка (FAB) - автоматически позиционируется
        floatingActionButton = {
            FloatingActionButton(
                onClick = onAddClick,
                containerColor = MaterialTheme.colorScheme.primary
            ) {
                Icon(
                    imageVector = androidx.compose.material.icons.Icons.Default.Add,
                    contentDescription = "Добавить"
                )
            }
        },
        
        // Основной контент
        content = { paddingValues ->
            LazyColumn(
                modifier = Modifier.fillMaxSize(),
                contentPadding = paddingValues 
            ) {
                items(habits, key = { it.id }) { habit ->
                    HabitCard(
                        habit = habit,
                        onClick = { onHabitClick(habit.id) }
                    )
                }
            }
        }
    )
}
```

## 6. Практика: Настройка навигации

Теперь свяжем экраны вместе. Создайте файл `commonMain/kotlin/ui/navigation/NavGraph.kt`.

```kotlin
package com.ecotrack.ui.navigation

import androidx.compose.runtime.Composable
import androidx.navigation.NavHostController
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable

// Маршруты
sealed class Screen() {
    object Home : Screen
    object AddHabit : Screen
}

@Composable
fun AppNavHost(
    navController: NavHostController,
    startDestination: Screen = Screen.Home
) {
    NavHost(
        navController = navController,
        startDestination = startDestination
    ) {
        composable<Screen.Home> {
            // Здесь будет HomeScreen (из модуля 1 мы его уже писали, теперь обновим)
            // Для примера оставим заглушку, но в задании нужно подключить HomeScreen из шага 3
            androidx.compose.material3.Text("Главный экран") 
        }

        composable<Screen.AddHabit> {
            // Экран добавления (реализуем в задании)
            androidx.compose.material3.Text("Экран добавления")
        }
    }
}
```

### Подключение навигации в MainActivity (Android)

Откройте `src/androidMain/kotlin/com/ecotrack/MainActivity.kt`:

```kotlin
package com.ecotrack

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.ui.Modifier
import androidx.navigation.compose.rememberNavController

// Импортируйте общий экран и навигацию
import com.ecotrack.ui.navigation.AppNavHost

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        setContent {
            MaterialTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    // Создаем NavController и передаем в NavHost
                    val navController = rememberNavController()
                    AppNavHost(navController = navController)
                }
            }
        }
    }
}
```

### Подключение навигации в MainViewController (iOS)

Откройте `src/iosMain/kotlin/com/ecotrack/MainViewController.kt`:

```kotlin
package com.ecotrack

import androidx.compose.ui.window.ComposeUIViewController
import androidx.navigation.compose.rememberNavController

// Импортируйте навигацию
import com.ecotrack.ui.navigation.AppNavHost

fun MainViewController() = ComposeUIViewController { 
    val navController = rememberNavController()
    AppNavHost(navController = navController)
}
```

---

## 📝 Домашнее задание (Модуль 2)

Вам нужно собрать работающий прототип навигации и UI.

### Задание 1: Подключите библиотеку навигации
- Выберите один из вариантов (Gradle Kotlin DSL или Version Catalog)
- Добавьте зависимость `androidx.navigation:navigation-compose:2.7.6` 

### Задание 2: Создайте экран добавления (`AddHabitScreen`) с Scaffold
- Используйте `Scaffold` как основной контейнер
- Добавьте `TopAppBar` с заголовком "Новая привычка" и кнопкой «Назад»
- Используйте `Column` внутри `content` Scaffold с правильными отступами (`paddingValues`)
- Добавьте `TextField` (для названия), `ExposedDropdownMenuBox` (или простой `Button` для выбора категории)
- Добавьте кнопку «Сохранить»

### Задание 3: Свяжите навигацию
- В `MainActivity.kt` (Android) и `MainViewController.kt` (iOS) создайте `NavController`
- Передайте его в `AppNavHost`
- Настройте переходы: с Главной на Добавление (через FAB) и обратно
- Убедитесь, что кнопка «Назад» в TopAppBar работает корректно

### Задание 4: Реализуйте список
- В `HomeScreen` используйте `Scaffold` с `TopAppBar` и `FAB`
- Создайте список из 3-х фиктивных объектов `Habit` в `LazyColumn`
- **Важно:** Передавайте `contentPadding = paddingValues` в LazyColumn!
- Убедитесь, что список не перекрывается системными панелями на iOS (notch, home indicator)

### Задание 5: Темизация
- Добавьте переключатель (Switch) в `HomeScreen` (можно добавить в TopAppBar или отдельным элементом)
- При нажатии меняйте переменную `darkTheme` в `EcoTrackTheme`
- Фон должен меняться с белого на черный

### Задание 6 (Бонус): Проверка отступов
- Запустите приложение на iOS симуляторе с Notch (iPhone 14/15)
- Убедитесь, что контент не перекрывается вырезом камеры
- Запустите на Android эмуляторе с навигационными кнопками
- Убедитесь, что контент не перекрывается нижней панелью

**Критерий сдачи:**
- ✅ Приложение запускается на Android и iOS.
- ✅ Есть список из 3 карточек с правильными отступами.
- ✅ При нажатии на FAB открывается экран добавления.
- ✅ Кнопка «Назад» работает корректно (возвращает на список).
- ✅ Переключатель темы меняет цвета интерфейса.
- ✅ **Контент не перекрывается системными панелями на iOS (notch, home indicator).**

---

**💡 Советы:**
1. Не пытайтесь сразу делать идеальную верстку. Сначала заставьте кнопки нажиматься и экраны переключаться, потом занимайтесь отступами (padding) и цветами.
2. Всегда используйте `Scaffold` для основных экранов — это сэкономит вам много времени на отладку отступов.
3. Тестируйте приложение на разных устройствах (с notch и без) — отступы могут отличаться!

Удачи! В следующем модуле мы начнем внедрять архитектуру (ViewModel) и отделим логику от UI.

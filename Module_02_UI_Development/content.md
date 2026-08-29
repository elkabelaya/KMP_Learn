# 📘 Модуль 2: UI-разработка на Compose Multiplatform

**Добро пожаловать во второй модуль!**  
В первом модуле мы запустили «Hello World». Теперь пришло время превратить это в настоящее приложение. Мы научимся создавать списки, переходить между экранами и настраивать темы (светлую/темную).

**Цели модуля:**
1. Освоить основные компоновки Compose (Column, Row, Box) и модификаторы
2. Научиться создавать эффективные списки (`LazyColumn`)
3. Реализовать навигацию между экранами (Navigation Compose)
4. Настроить тему приложения (Material3) и переключение цветов

**Время выполнения:** ~8–10 часов.

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

В KMP мы используем библиотеку **`androidx.navigation.compose`**. Она работает одинаково на Android и iOS.

### 🔗 Подключение библиотеки навигации

**⚠️** В Kotlin Multiplatform проектах зависимости должны добавляться **в соответствующих sourceSets** внутри блока `kotlin { ... }`, а не в обычный блок `dependencies`.

#### Вариант 1: Gradle Kotlin DSL (`.gradle.kts`) — Рекомендуемый

Откройте файл `shared/build.gradle.kts` и добавьте зависимости **внутри блока kotlin**:

```kotlin
dependencies {
    commonMainImplementation("androidx.navigation:navigation-compose:2.7.6")
}
```
или 
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

**Почему это важно:**
- `commonMainImplementation` — зависимость будет доступна в общем коде (работает на всех платформах)
- `androidMainImplementation` — только для Android
- `iosMainImplementation` — только для iOS

#### Вариант 2: Gradle Version Catalog (`libs.versions.toml`) — Современный подход

Если вы используете **Version Catalog** (рекомендуется для новых проектов), создайте или отредактируйте файл `gradle/libs.versions.toml` в корне проекта:

```toml
[versions]
navigation-compose = "2.7.6"

[libraries]
navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigation-compose" }
```

Затем в `shared/build.gradle.kts` используйте алиас **внутри sourceSets**:

```kotlin
kotlin {
    sourceSets {
        commonMain.dependencies {
            // ✅ Используем алиас из Version Catalog
            implementation(libs.navigation.compose)
        }
    }
}
```

**Преимущества Version Catalog:**
- Централизованное управление версиями зависимостей
- Легко обновлять версии в одном месте
- Избегает дублирования версий

### Основные понятия навигации:
1. **`NavHost`**: Главный контейнер, который управляет экранами
2. **Route (Маршрут)**: Строка-идентификатор экрана (например, `"home"` или `"addHabit/{id}"`)
3. **`NavController`**: Объект, который переключает экраны (`navController.navigate("addHabit")`)

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

## 4. Практика: Реализация списка привычек

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
import androidx.compose.material3.FloatingActionButton
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier

@Composable
fun HomeScreen(
    habits: List<com.ecotrack.data.model.Habit>, // Данные из ViewModel (пока заглушка)
    onAddClick: () -> Unit,
    onHabitClick: (String) -> Unit // ID привычки
) {
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(bottom = 80.dp) // Отступ под FAB
    ) {
        item {
            Text(
                text = "Мои привычки",
                style = MaterialTheme.typography.headlineMedium,
                modifier = Modifier.padding(16.dp)
            )
        }

        items(habits, key = { it.id }) { habit ->
            HabitCard(
                habit = habit,
                onClick = { onHabitClick(habit.id) }
            )
        }
    }

    // Плавающая кнопка (FAB)
    FloatingActionButton(
        onClick = onAddClick,
        modifier = Modifier
            .align(Alignment.CenterHorizontally)
            .padding(bottom = 16.dp)
    ) {
        Icon(
            imageVector = androidx.compose.material.icons.Icons.Default.Add,
            contentDescription = "Добавить"
        )
    }
}
```

---

## 5. Практика: Настройка навигации

Теперь свяжем экраны вместе. Создайте файл `commonMain/kotlin/ui/navigation/NavGraph.kt`.

```kotlin
package com.ecotrack.ui.navigation

import androidx.compose.runtime.Composable
import androidx.navigation.NavHostController
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable

// Маршруты
sealed class Screen(val route: String) {
    object Home : Screen("home")
    object AddHabit : Screen("add_habit")
}

@Composable
fun AppNavHost(
    navController: NavHostController,
    startDestination: String = Screen.Home.route
) {
    NavHost(
        navController = navController,
        startDestination = startDestination
    ) {
        composable(Screen.Home.route) {
            // Здесь будет HomeScreen (из модуля 1 мы его уже писали, теперь обновим)
            // Для примера оставим заглушку, но в задании нужно подключить HomeScreen из шага 3
            androidx.compose.material3.Text("Главный экран") 
        }

        composable(Screen.AddHabit.route) {
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

### Задание 2: Создайте экран добавления (`AddHabitScreen`)
- Используйте `Column`, `TextField` (для названия), `ExposedDropdownMenuBox` (или простой `Button` для выбора категории)
- Добавьте кнопку «Сохранить»

### Задание 3: Свяжите навигацию
- В `MainActivity.kt` (Android) и `MainViewController.kt` (iOS) создайте `NavController`
- Передайте его в `AppNavHost`
- Настройте переходы: с Главной на Добавление (через FAB) и обратно

### Задание 4: Реализуйте список
- В `HomeScreen` создайте список из 3-х фиктивных объектов `Habit`
- Убедитесь, что список прокручивается (LazyColumn)

### Задание 5: Темизация
- Добавьте переключатель (Switch) в `HomeScreen`
- При нажатии меняйте переменную `darkTheme` в `EcoTrackTheme`
- Фон должен меняться с белого на черный

**Критерий сдачи:**
- Приложение запускается.
- Есть список из 3 карточек.
- При нажатии на FAB открывается экран добавления.
- Кнопка «Назад» работает корректно (возвращает на список).
- Переключатель темы меняет цвета интерфейса.

---

**💡 Совет:** Не пытайтесь сразу делать идеальную верстку. Сначала заставьте кнопки нажиматься и экраны переключаться, потом занимайтесь отступами (padding) и цветами.

Удачи! В следующем модуле мы начнем внедрять архитектуру (ViewModel) и отделим логику от UI.

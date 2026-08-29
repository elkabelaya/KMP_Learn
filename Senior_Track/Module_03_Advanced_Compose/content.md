# 📘 Модуль 2: Продвинутый Compose Multiplatform

**Добро пожаловать во второй модуль Senior Track!**  
В этом модуле мы выйдем за рамки базовых компонентов Compose. Вы научитесь создавать кастомные Layout'ы, оптимизировать производительность и расширять приложение на Desktop и Web платформы.

**Цели модуля:**
1. Освоить создание кастомных Layout-компонентов
2. Научиться оптимизировать производительность Compose UI
3. Реализовать сложные анимации и переходы
4. Добавить Desktop-версию SkillSync с нативными фичами
5. Опубликовать Web-версию через Compose Web

**Время выполнения:** ~30–40 часов (6 недель).

---

## 1. Введение: Ограничения базовых компонентов

В базовом курсе мы использовали `Column`, `Row`, `Box`. Но что делать, когда:
- Нужно реализовать Masonry Grid (как в Pinterest)?
- Требуется сложная сетка с элементами разной высоты?
- Стандартные компоненты не дают нужной гибкости?

**Решение:** Кастомные Layout-ы через `Layout` компонент.

---

## 2. Теория: Кастомные Layout-ы в Compose

### Как работает Layout?

Компонент `Layout` дает полный контроль над размещением дочерних элементов.

```kotlin
@Composable
fun CustomLayout(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        modifier = modifier,
        content = content
    ) { measurables, constraints ->
        // 1. Измеряем дочерные элементы
        val placeables = measurables.map { measurable ->
            measurable.measure(constraints)
        }

        // 2. Вычисляем размеры самого Layout-а
        val totalWidth = placeables.sumOf { it.width }
        val totalHeight = placeables.maxOfOrNull { it.height } ?: 0

        layout(totalWidth, totalHeight) {
            // 3. Размещаем элементы
            var left = 0
            placeables.forEach { placeable ->
                val right = left + placeable.width
                placeable.place(left, 0)
                left = right
            }
        }
    }
}
```

### Ключевые концепции:

#### 📏 Constraints (Ограничения)
```kotlin
data class Constraints(
    val minWidth: Int,
    val maxWidth: Int,
    val minHeight: Int,
    val maxHeight: Int
)
```

#### 📐 Placeable (Измеренный элемент)
```kotlin
interface Placeable {
    val width: Int
    val height: Int
    fun place(x: Int, y: Int)
}
```

#### 🎯 MeasurePolicy (Политика измерения)
Лямбда, которая определяет как измерять и размещать элементы.

---

## 3. Практика: Создание Masonry Grid для SkillSync

### Задача
Создать сетку навыков, где карточки имеют разную высоту (как в Pinterest).

### Шаг 1: Создание MasonryLayout компонента

Создайте `shared/core/core-ui/src/commonMain/kotlin/layout/MasonryLayout.kt`:

```kotlin
package layout

import androidx.compose.runtime.*
import androidx.compose.ui.layout.*
import androidx.compose.ui.Modifier

@Composable
fun MasonryLayout(
    columns: Int = 2,
    spacing: Int = 16,
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        modifier = modifier,
        content = content
    ) { measurables, constraints ->
        // Проверяем, что есть место для колонок
        if (constraints.maxWidth < columns * 100) {
            // Если места мало, используем одну колонку
        }

        // 1. Измеряем все элементы
        val placeables = measurables.map { measurable ->
            // Делим доступную ширину на количество колонок
            val columnConstraints = constraints.copy(
                maxWidth = (constraints.maxWidth / columns) - spacing
            )
            measurable.measure(columnConstraints)
        }

        // 2. Распределяем по колонкам
        val columnsHeights = MutableList(columns) { 0 }
        val columnItems = List(columns) { mutableListOf<Placeable>() }

        placeables.forEachIndexed { index, placeable ->
            // Находим колонку с минимальной высотой
            val columnIndex = columnsHeights.indexOfMin()
            
            columnItems[columnIndex].add(placeable)
            columnsHeights[columnIndex] += placeable.height + spacing
        }

        // 3. Вычисляем общие размеры
        val totalWidth = constraints.maxWidth
        val totalHeight = columnsHeights.maxOrNull() ?: 0

        layout(totalWidth, totalHeight) {
            // 4. Размещаем элементы по колонкам
            val columnWidth = (totalWidth - (columns - 1) * spacing) / columns
            
            columnItems.forEachIndexed { columnIndex, items ->
                var yOffset = 0
                val xOffset = columnIndex * (columnWidth + spacing)

                items.forEach { placeable ->
                    placeable.place(xOffset, yOffset)
                    yOffset += placeable.height + spacing
                }
            }
        }
    }
}

// Extension функция для поиска минимума
fun List<Int>.indexOfMin(): Int {
    var minIndex = 0
    var minValue = this[0]
    for (i in 1 until size) {
        if (this[i] < minValue) {
            minValue = this[i]
            minIndex = i
        }
    }
    return minIndex
}
```

### Шаг 2: Использование MasonryLayout в каталоге навыков

Создайте `shared/feature/feature-skills/src/commonMain/kotlin/screens/SkillsCatalogScreen.kt`:

```kotlin
package screens

import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import layout.MasonryLayout
import components.SkillCard

@Composable
fun SkillsCatalogScreen(
    skills: List<Skill>,
    onSkillClick: (String) -> Unit
) {
    MasonryLayout(
        columns = 2,
        spacing = 16,
        modifier = Modifier.fillMaxSize()
    ) {
        skills.forEach { skill ->
            SkillCard(
                skill = skill,
                onClick = { onSkillClick(skill.id) }
            )
        }
    }
}
```

---

## 4. Теория: Оптимизация производительности Compose

### Проблемы производительности

1. **Излишние перерисовки (Recomposition)**
2. **Неэффективное использование remember**
3. **Тяжелые вычисления в @Composable функциях**

### 🚫 Антипаттерн: Вычисления в @Composable

```kotlin
// ПЛОХО: Вычисляется при каждой перерисовке
@Composable
fun ExpensiveScreen(data: List<Item>) {
    val sortedData = data.sortedBy { it.name } // ❌ Вычисляется каждый раз!
    
    LazyColumn {
        items(sortedData) { item -> /* ... */ }
    }
}
```

### ✅ Паттерн: remember с ключом

```kotlin
// ХОРОШО: Вычисляется только когда data меняется
@Composable
fun OptimizedScreen(data: List<Item>) {
    val sortedData = remember(data) { 
        data.sortedBy { it.name } // ✅ Кэшируется
    }
    
    LazyColumn {
        items(sortedData) { item -> /* ... */ }
    }
}
```

### 🎯 derivedStateOf для производных состояний

```kotlin
@Composable
fun SearchScreen(items: List<Item>, query: String) {
    // Фильтр вычисляется только когда результат меняется
    val filteredItems = derivedStateOf { 
        if (query.isEmpty()) items 
        else items.filter { it.name.contains(query, ignoreCase = true) }
    }.value
    
    LazyColumn {
        items(filteredItems) { item -> /* ... */ }
    }
}
```

---

## 5. Практика: Оптимизация списка с 10k+ элементов

### Задача
Создать список, который плавно прокручивается с 10,000 элементов.

### Шаг 1: Настройка LazyColumn с предзагрузкой

```kotlin
@Composable
fun OptimizedListScreen(items: List<Item>) {
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        // Загружаем элементы до и после видимой области
        ensureVisibleParameters = EnsureVisibleParams(
            animationSpec = tween(300)
        ),
        // Предзагружаем 10 элементов вперед
        userScrollState = rememberLazyListState()
    ) {
        items(
            items, 
            key = { it.id }, // ✅ Всегда используйте ключи!
            itemContent = { item ->
                // Мемоизируем компонент карточки
                remember(item) { 
                    ItemCard(item = item) 
                }
            }
        )
    }
}
```

### Шаг 2: Профилирование с Layout Inspector

1. Запустите приложение с флагом `-Pcompose.experimental.enableLayoutInspector=true`
2. Откройте Layout Inspector в Android Studio
3. Найдите компоненты с частыми перерисовками
4. Оптимизируйте через `remember` и `derivedStateOf`

---

## 6. Теория: Анимации в Compose

### Типы анимаций:

#### 🎬 AnimatedVisibility
Появление/исчезновение элементов.

```kotlin
@Composable
fun AnimatedItem(visible: Boolean) {
    AnimatedVisibility(
        visible = visible,
        enter = expandVertically(),
        exit = shrinkVertically()
    ) {
        Card { /* ... */ }
    }
}
```

#### 🔄 Transition
Сложные переходы между состояниями.

```kotlin
@Composable
fun TransitionScreen(mode: ScreenMode) {
    val crossfade = updateTransition(mode, label = "modeTransition")
    
    Crossfade(
        targetState = mode,
        animationSpec = tween(300)
    ) { currentMode ->
        when (currentMode) {
            ScreenMode.List -> ListScreen()
            ScreenMode.Grid -> GridScreen()
        }
    }
}
```

#### 🎨 AnimatedContent
Анимация при смене контента.

```kotlin
@Composable
fun AnimatedContentScreen(selectedId: String) {
    AnimatedContent(
        selectedId,
        transitionSpec = {
            slideIntoContainer(
                towards = AnimatedContent.Towards.Start,
                animationSpec = tween(300)
            ) togetherWith
            slideOutOfContainer(
                towards = AnimatedContent.Towards.End,
                animationSpec = tween(300)
            )
        }
    ) { id ->
        DetailScreen(itemId = id)
    }
}
```

---

## 7. Практика: Добавление Desktop-версии SkillSync

### Шаг 1: Настройка Desktop таргета

В `shared/build.gradle.kts` добавьте:

```kotlin
kotlin {
    // ... существующие таргеты
    
    jvm() {
        withJava()
        
        compilations.all {
            kotlinOptions.jvmTarget = "17"
        }
        
        compilations["main"].defaultSourceSet {
            dependencies {
                implementation("org.jetbrains.compose.desktop:desktop:1.5.0")
            }
        }
    }
}
```

### Шаг 2: Создание Desktop приложения

Создайте `apps/skillsync-desktop/build.gradle.kts`:

```kotlin
plugins {
    kotlin("multiplatform")
    id("org.jetbrains.compose.desktop.application") version "1.5.0"
}

kotlin {
    jvm()
    
    sourceSets {
        val jvmMain by getting {
            dependencies {
                implementation(project(":shared"))
                implementation(compose.desktop.currentOs)
            }
        }
    }
}

compose.desktop {
    application {
        mainClass = "MainKt"
        
        nativeDistributions {
            modules("java.sql")
            
            packageName = "SkillSync"
            packageVersion = "1.0.0"
            
            // macOS
            mac {
                bundleID = "com.skillsync.desktop"
                iconFile.set(file("icons/app_icon.icns"))
            }
            
            // Windows
            windows {
                iconFile.set(file("icons/app_icon.ico"))
                moduleName = "skillsync"
            }
            
            // Linux
            linux {
                iconFile.set(file("icons/app_icon.png"))
            }
        }
    }
}
```

### Шаг 3: Main функция для Desktop

Создайте `apps/skillsync-desktop/src/jvmMain/kotlin/Main.kt`:

```kotlin
import androidx.compose.desktop.ui.tooling.preview.Preview
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.compose.desktop.ui.tooling.DesktopWindowInfo
import org.jetbrains.compose.desktop.application.dsl.StatsConsent
import org.jetbrains.compose.desktop.ui.tooling.preview.PreviewParameter

fun main() = application {
    window(
        title = "SkillSync Desktop",
        width = 1200.dp,
        height = 800.dp
    ) {
        SkillSyncApp()
    }
}

@Composable
@Preview
fun SkillSyncApp() {
    MaterialTheme {
        Surface(
            modifier = Modifier.fillMaxSize(),
            color = MaterialTheme.colorScheme.background
        ) {
            // Ваш основной UI из commonMain
            AppNavHost()
        }
    }
}
```

### Шаг 4: Нативные фичи Desktop (меню, окна)

Создайте `shared/src/jvmMain/kotlin/platform/DesktopPlatform.kt`:

```kotlin
package platform

import androidx.compose.runtime.*
import androidx.compose.ui.platform.LocalWindowInfo
import org.jetbrains.compose.desktop.ui.tooling.DesktopWindowInfo

actual fun getPlatformName(): String = "Desktop"

@Composable
actual fun PlatformSpecificFeatures() {
    val windowInfo = LocalWindowInfo.current
    
    // Доступ к нативным фичам Desktop
    LaunchedEffect(Unit) {
        // Можно управлять окном, меню и т.д.
    }
}
```

---

## 8. Практика: Публикация Web-версии через Compose Web

### Шаг 1: Настройка Web таргета

В `shared/build.gradle.kts`:

```kotlin
kotlin {
    js(IR) {
        browser()
        binaries.executable()
        
        compilations.all {
            kotlinOptions {
                moduleKind = "umd"
                metaInfo = true
            }
        }
    }
}
```

### Шаг 2: Создание Web приложения

Создайте `apps/skillsync-web/build.gradle.kts`:

```kotlin
plugins {
    kotlin("multiplatform")
}

kotlin {
    js(IR) {
        browser()
        binaries.executable().apply {
            testRun {
                dependencies {
                    implementation(project(":shared"))
                }
            }
        }
    }
    
    sourceSets {
        val jsMain by getting {
            dependencies {
                implementation(project(":shared"))
            }
        }
    }
}

tasks {
    val browserDev by creating(JavaExec::class) {
        group = "verification"
        classpath(sourceSets["jsMain"].runtimeClasspath)
        main = "SkillSyncMain_kotlin.js"
    }
}
```

### Шаг 3: Публикация на GitHub Pages

1. Соберите Web-версию: `./gradlew :apps:skillsync-web:jsBrowserProductionWebpack`
2. Файлы будут в `apps/skillsync-web/build/dist/compileSync/js/mainProduction`
3. Загрузите в репозиторий и включите GitHub Pages

---

## 📝 Домашнее задание (Модуль 2)

### Задача: Реализовать продвинутый UI для SkillSync

**Требования:**

1. **Создайте кастомный Layout:**
   - Реализуйте Masonry Grid для каталога навыков
   - Добавьте адаптивное количество колонок (1 на mobile, 2-3 на desktop)

2. **Оптимизация производительности:**
   - Профилируйте список с 10k элементов через Layout Inspector
   - Устраните излишние перерисовки (целевая метрика: 60 FPS при прокрутке)
   - Используйте `remember`, `derivedStateOf` корректно

3. **Анимации:**
   - Добавьте анимацию появления карточек в списке (AnimatedVisibility)
   - Реализуйте переход между List/Grid режимами (Crossfade)
   - Добавьте анимацию при нажатии на карточку (scale effect)

4. **Desktop-версия:**
   - Настройте запуск на Desktop (macOS/Windows/Linux)
   - Добавьте нативное меню (File, Edit, Help)
   - Реализуйте управление окном (maximize, minimize, close)

5. **Web-версия:**
   - Настройте сборку для Web
   - Опубликуйте на GitHub Pages или Vercel
   - Убедитесь, что работает в Chrome, Firefox, Safari

**Критерии сдачи:**
- ✅ Masonry Grid работает плавно (60 FPS)
- ✅ Список с 10k элементов прокручивается без лагов
- ✅ Анимации плавные и не вызывают мерцания
- ✅ Desktop-версия запускается на вашей ОС
- ✅ Web-версия доступна по публичной ссылке

**Бонусные задания:**
- Реализуйте кастомный анимированный переход между экранами
- Добавьте gesture-based navigation (свайпы для навигации)
- Создайте кастомный компонент с сложной анимацией (например, ленивый список с pull-to-refresh)

---

## 💡 Советы по выполнению

1. **Начните с оптимизации:** Сначала убедитесь, что базовый список работает плавно, потом добавляйте анимации.
2. **Используйте Layout Inspector:** Это ваш главный инструмент для отладки производительности.
3. **Тестируйте на реальном устройстве:** Эмуляторы могут скрывать проблемы с производительностью.
4. **Не переусердствуйте с анимациями:** Пользователи ценят плавность, но не любят лишние эффекты.
5. **Desktop и Web — это бонус:** Если возникнут сложности, сосредоточьтесь на Android/iOS.

---

**Следующий модуль:** В Module_03_Advanced_Data мы изучим продвинутую работу с данными: GraphQL, Real-time синхронизацию и сложные стратегии кэширования.

Удачи! 🚀

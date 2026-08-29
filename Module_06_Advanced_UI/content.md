# 📘 Модуль 6: Продвинутый UI и Визуализация

**Добро пожаловать в шестой модуль!**  
Теперь, когда у нас есть данные и синхронизация, пришло время сделать приложение красивым. Мы добавим графики статистики, анимации и кастомные компоненты.

**Цели модуля:**
1. Интегрировать библиотеку **Vico** для графиков в KMP
2. Реализовать анимации переходов и состояний
3. Создать кастомные компоненты (бейджи, прогресс-бары)
4. Настроить дизайн-систему (цвета, шрифты в commonMain)

**Время выполнения:** ~10–12 часов.

---

## 1. Теория: Графики в Compose Multiplatform

### Варианты графиков для KMP:

| Библиотека | Плюсы | Минусы | Рекомендация |
|------------|-------|--------|--------------|
| **Vico** | Нативный Compose, KMP-поддержка | Меньше типов графиков | ✅ **Рекомендуется** |
| MPAndroidChart | Много типов графиков | Сложная интеграция с Compose | Для сложных дашбордов |
| Canvas API | Полный контроль | Нужно рисовать всё вручную | Для кастомных визуализаций |

### Почему Vico?
- **Compose-first:** Полная интеграция с Compose Multiplatform
- **Производительность:** Оптимизирован для 60 FPS
- **Кастомизация:** Полный контроль над стилями

---

## 2. Практика: Интеграция Vico для графиков

### Шаг 1: Добавляем зависимости

**Файл:** `shared/build.gradle.kts`

```kotlin
dependencies {
    // Vico для графиков (KMP-совместимая версия)
    implementation("com.mikepenz:multiplatform-settings-ui:1.2.0") // Для настроек
    
    // Альтернатива: использовать Canvas для простых графиков
    // или создать кастомный компонент на Canvas API
}

// Для Vico пока нет полной KMP-поддержки, поэтому используем Canvas
```

**Примечание:** На момент написания курса Vico не имеет полной KMP-поддержки. Мы создадим кастомный график на Canvas API, который будет работать везде.

### Шаг 2: Создаем кастомный график на Canvas

**Файл:** `src/commonMain/kotlin/com/ecotrack/ui/components/ActivityChart.kt`

```kotlin
package com.ecotrack.ui.components

import androidx.compose.canvas.draw
import androidx.compose.foundation.Canvas
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.geometry.Size
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.drawscope.Stroke
import androidx.compose.ui.unit.dp

@Composable
fun ActivityChart(
    dataPoints: List<Float>,  // Значения активности за N дней
    modifier: Modifier = Modifier,
    lineColor: Color = Color(0xFF4CAF50),
    pointColor: Color = Color(0xFF81C784),
    showPoints: Boolean = true,
    maxPoints: Int = 7  // Количество точек (например, неделя)
) {
    Canvas(
        modifier = modifier
            .fillMaxWidth()
            .height(200.dp)
    ) {
        if (dataPoints.isEmpty()) return@Canvas
        
        val size = Size(size.width, size.height)
        val padding = 40f
        val chartWidth = size.width - (padding * 2)
        val chartHeight = size.height - (padding * 2)
        
        // Находим максимальное значение для масштабирования
        val maxValue = dataPoints.maxOrNull() ?: 1f
        
        // Рисуем сетку
        drawContext.drawRect(
            color = Color.LightGray.copy(alpha = 0.3f),
            topLeft = Offset(padding, padding),
            size = Size(chartWidth, chartHeight),
            style = Stroke(width = 1f)
        )
        
        // Вычисляем координаты точек
        val points = mutableListOf<Offset>()
        val stepX = chartWidth / (maxPoints - 1)
        
        dataPoints.forEachIndexed { index, value ->
            val x = padding + (index * stepX)
            val y = size.height - padding - ((value / maxValue) * chartHeight)
            points.add(Offset(x, y))
        }
        
        // Рисуем линию графика
        if (points.size > 1) {
            drawContext.drawPolyline(
                color = lineColor,
                points = points.toTypedArray(),
                style = Stroke(width = 3f)
            )
        }
        
        // Рисуем точки
        if (showPoints) {
            points.forEach { point ->
                drawContext.drawCircle(
                    color = pointColor,
                    radius = 6f,
                    center = point
                )
            }
        }
        
        // Рисуем оси
        drawContext.drawLine(
            color = Color.Gray,
            start = Offset(padding, size.height - padding),
            end = Offset(size.width - padding, size.height - padding),
            strokeWidth = 2f
        )
        
        drawContext.drawLine(
            color = Color.Gray,
            start = Offset(padding, padding),
            end = Offset(padding, size.height - padding),
            strokeWidth = 2f
        )
    }
}

// Экран статистики с графиком
@Composable
fun StatisticsScreen(
    weeklyData: List<Float> = listOf(5f, 8f, 12f, 7f, 15f, 10f, 18f),
    monthlyData: List<Float> = listOf(35f, 42f, 38f, 45f)
) {
    androidx.compose.foundation.layout.Column(
        modifier = androidx.compose.foundation.layout.Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        androidx.compose.material3.Text(
            text = "Статистика",
            style = androidx.compose.material3.MaterialTheme.typography.headlineMedium,
        )
        
        androidx.compose.foundation.layout.Spacer(
            modifier = androidx.compose.foundation.layout.Modifier.height(24.dp)
        )
        
        // График за неделю
        androidx.compose.material3.Text(
            text = "Активность за неделю",
            style = androidx.compose.material3.MaterialTheme.typography.titleMedium
        )
        
        ActivityChart(
            dataPoints = weeklyData,
            modifier = androidx.compose.foundation.layout.Modifier
                .fillMaxWidth()
                .padding(vertical = 16.dp)
        )
        
        androidx.compose.foundation.layout.Spacer(
            modifier = androidx.compose.foundation.layout.Modifier.height(32.dp)
        )
        
        // График за месяц
        androidx.compose.material3.Text(
            text = "Активность за месяц",
            style = androidx.compose.material3.MaterialTheme.typography.titleMedium
        )
        
        ActivityChart(
            dataPoints = monthlyData,
            modifier = androidx.compose.foundation.layout.Modifier
                .fillMaxWidth()
                .padding(vertical = 16.dp)
        )
    }
}
```

---

## 3. Практика: Анимации в Compose

### Типы анимаций:

1. **AnimatedVisibility** — показ/скрытие с анимацией
2. **updateTransition** — плавное изменение параметров
3. **animate*AsState** — анимация значений

### Шаг 1: Анимированная карточка бейджа

**Файл:** `src/commonMain/kotlin/com/ecotrack/ui/components/AchievementCard.kt`

```kotlin
package com.ecotrack.ui.components

import androidx.compose.animation.*
import androidx.compose.animation.core.tween
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.scale
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight

data class Achievement(
    val id: String,
    val title: String,
    val description: String,
    val isUnlocked: Boolean,
    val progress: Float = 0f // 0.0 - 1.0
)

@Composable
fun AchievementCard(
    achievement: Achievement,
    onClick: () -> Unit = {}
) {
    var isPressed by remember { mutableStateOf(false) }
    
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(vertical = 8.dp)
            .animateContentSize(tween(300))
            .scale(scale = if (isPressed) 0.95f else 1f)
            .animateItemPlacement(tween(300))
            .clickable(onClick = { 
                isPressed = true
                onClick()
                isPressed = false
            }),
        colors = CardDefaults.cardColors(
            containerColor = if (achievement.isUnlocked) 
                Color(0xFFE8F5E9) else MaterialTheme.colorScheme.surfaceVariant
        )
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            // Иконка с анимацией
            Box(
                modifier = Modifier
                    .size(48.dp)
                    .animateItemOpacity(tween(500))
            ) {
                Icon(
                    imageVector = androidx.compose.material.icons.Icons.Default.Achievement,
                    contentDescription = null,
                    tint = if (achievement.isUnlocked) 
                        Color(0xFF4CAF50) else Color.Gray,
                    modifier = Modifier.size(48.dp)
                )
            }
            
            Spacer(modifier = Modifier.width(16.dp))
            
            Column {
                Text(
                    text = achievement.title,
                    style = MaterialTheme.typography.titleMedium,
                    fontWeight = FontWeight.Bold,
                    color = if (achievement.isUnlocked) 
                        MaterialTheme.colorScheme.onSurface else Color.Gray
                )
                
                Text(
                    text = achievement.description,
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
                
                // Прогресс-бар (если не разблокирован)
                if (!achievement.isUnlocked && achievement.progress > 0) {
                    Spacer(modifier = Modifier.height(8.dp))
                    
                    LinearProgressIndicator(
                        progress = achievement.progress,
                        modifier = Modifier.fillMaxWidth(),
                        color = Color(0xFF4CAF50),
                        trackColor = Color.Gray.copy(alpha = 0.3f)
                    )
                }
            }
        }
    }
}
```

### Шаг 2: Анимация переходов между экранами

**Файл:** `src/commonMain/kotlin/com/ecotrack/ui/navigation/AnimatedNavHost.kt`

```kotlin
package com.ecotrack.ui.navigation

import androidx.compose.animation.*
import androidx.compose.animation.core.tween
import androidx.compose.runtime.Composable
import androidx.navigation.NavHostController
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable

@Composable
fun AnimatedNavHost(
    navController: NavHostController,
    startDestination: String
) {
    NavHost(
        navController = navController,
        startDestination = startDestination,
        enterTransition = { 
            slideIntoContainer(
                AnimatedContentTransitionScope.SlideDirection.Left, 
                tween(300)
            ) + fadeIn(tween(300))
        },
        exitTransition = { 
            slideOutOfContainer(
                AnimatedContentTransitionScope.SlideDirection.Left, 
                tween(300)
            ) + fadeOut(tween(300))
        },
        popEnterTransition = { 
            slideIntoContainer(
                AnimatedContentTransitionScope.SlideDirection.Right, 
                tween(300)
            ) + fadeIn(tween(300))
        },
        popExitTransition = { 
            slideOutOfContainer(
                AnimatedContentTransitionScope.SlideDirection.Right, 
                tween(300)
            ) + fadeOut(tween(300))
        }
    ) {
        composable(Screen.Home.route) {
            HomeScreen(
                // ... параметры
            )
        }
        
        composable(Screen.AddHabit.route) {
            AddHabitScreen(
                // ... параметры
            )
        }
        
        composable(Screen.Statistics.route) {
            StatisticsScreen(
                // ... параметры
            )
        }
    }
}
```

---

## 4. Практика: Дизайн-система в commonMain

### Шаг 1: Создаем Colors.kt

**Файл:** `src/commonMain/kotlin/com/ecotrack/ui/theme/Colors.kt`

```kotlin
package com.ecotrack.ui.theme

import androidx.compose.ui.graphics.Color

// Светлая тема
val LightPrimary = Color(0xFF4CAF50)
val LightSecondary = Color(0xFF81C784)
val LightBackground = Color(0xFFF5F7FA)
val LightSurface = Color(0xFFFFFFFF)
val LightError = Color(0xFFF44336)

// Темная тема
val DarkPrimary = Color(0xFF81C784)
val DarkSecondary = Color(0xFF4CAF50)
val DarkBackground = Color(0xFF121212)
val DarkSurface = Color(0xFF1E1E1E)
val DarkError = Color(0xFFCF6679)

// Общие цвета
val Success = Color(0xFF00E676)
val Warning = Color(0xFFFFB74D)
val Info = Color(0xFF2196F3)
```

### Шаг 2: Создаем Typography.kt

**Файл:** `src/commonMain/kotlin/com/ecotrack/ui/theme/Typography.kt`

```kotlin
package com.ecotrack.ui.theme

import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.text.font.FontStyle
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.sp

object EcoTrackTypography {
    val displayLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Bold,
        fontSize = 32.sp,
        lineHeight = 40.sp,
        letterSpacing = (-0.5).sp
    )
    
    val displayMedium = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Bold,
        fontSize = 28.sp,
        lineHeight = 36.sp
    )
    
    val headlineLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.SemiBold,
        fontSize = 24.sp,
        lineHeight = 32.sp
    )
    
    val titleLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Medium,
        fontSize = 20.sp,
        lineHeight = 28.sp
    )
    
    val bodyLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp
    )
    
    val bodyMedium = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 14.sp,
        lineHeight = 20.sp
    )
    
    val labelSmall = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Medium,
        fontSize = 12.sp,
        lineHeight = 16.sp
    )
}
```

### Шаг 3: Создаем Theme.kt

**Файл:** `src/commonMain/kotlin/com/ecotrack/ui/theme/Theme.kt`

```kotlin
package com.ecotrack.ui.theme

import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.runtime.Immutable
import androidx.compose.ui.graphics.Color

@Immutable
data class EcoTrackColors(
    val primary: Color,
    val secondary: Color,
    val background: Color,
    val surface: Color,
    val error: Color
)

val LightEcoTrackColors = EcoTrackColors(
    primary = LightPrimary,
    secondary = LightSecondary,
    background = LightBackground,
    surface = LightSurface,
    error = LightError
)

val DarkEcoTrackColors = EcoTrackColors(
    primary = DarkPrimary,
    secondary = DarkSecondary,
    background = DarkBackground,
    surface = DarkSurface,
    error = DarkError
)

@Composable
fun EcoTrackTheme(
    darkTheme: Boolean = false,
    content: @Composable () -> Unit
) {
    val colors = if (darkTheme) DarkEcoTrackColors else LightEcoTrackColors
    
    MaterialTheme(
        colorScheme = if (darkTheme) {
            darkColorScheme(
                primary = colors.primary,
                secondary = colors.secondary,
                background = colors.background,
                surface = colors.surface,
                error = colors.error
            )
        } else {
            lightColorScheme(
                primary = colors.primary,
                secondary = colors.secondary,
                background = colors.background,
                surface = colors.surface,
                error = colors.error
            )
        },
        typography = Typography(
            displayLarge = EcoTrackTypography.displayLarge,
            headlineLarge = EcoTrackTypography.headlineLarge,
            titleLarge = EcoTrackTypography.titleLarge,
            bodyLarge = EcoTrackTypography.bodyLarge
        ),
        content = content
    )
}
```

---

## 📝 Домашнее задание (Модуль 6)

### Задание 1: Создайте экран статистики
- Используйте `ActivityChart` для отображения активности за неделю и месяц
- Добавьте легенду с описанием цветов

### Задание 2: Реализуйте систему достижений
- Создайте список из 5 бейджей (разные уровни прогресса)
- Добавьте анимацию разблокировки (scale + fadeIn)

### Задание 3: Настройте дизайн-систему
- Создайте файлы `Colors.kt`, `Typography.kt`, `Theme.kt` в commonMain
- Примените их во всех экранах

### Задание 4: Добавьте анимации переходов
- Настройте `AnimatedNavHost` для плавных переходов между экранами
- Добавьте анимацию нажатия кнопок (scale down)

### Задание 5: Адаптивность
- Убедитесь, что графики корректно отображаются на разных размерах экранов
- Добавьте поддержку landscape ориентации

**Критерий сдачи:**
- Графики отображаются корректно и плавно анимируются
- Бейджи имеют анимацию разблокировки
- Дизайн-система используется во всем приложении
- Переходы между экранами плавные

---

**💡 Совет:** Для отладки анимаций используйте **Developer Options** в Android Studio и включите "Show layout bounds" и "Debug GPU overlay".

**Важно:** Не переусердствуйте с анимациями. Они должны улучшать UX, а не отвлекать пользователя. Правило: если анимация длится более 300ms — она слишком медленная.

Удачи! В следующем модуле мы добавим нативные фичи: уведомления, deep links и работу с файловой системой.

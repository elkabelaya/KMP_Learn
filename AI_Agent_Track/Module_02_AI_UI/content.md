# Модуль 2: AI-помощник в разработке UI

## 📋 Обзор модуля

**Продолжительность:** 2 недели  
**Сложность:** Beginner → Intermediate  
**Цель:** Освоить генерацию Compose компонентов с контекстом проекта и создание дизайн-системы через AI

---

## 🎯 Цели обучения

После прохождения модуля вы сможете:
- ✅ Генерировать **Compose компоненты** из текстового описания или Figma макетов
- ✅ Создавать **дизайн-систему** (цвета, типографика, spacing) через AI
- ✅ Реализовывать **Material 3 темизацию** с кастомными темами
- ✅ Интегрировать **AI как фичу в приложение** (чат-боты, умный поиск)
- ✅ Оптимизировать **производительность UI** с помощью AI

---

## 📚 Темы модуля

### Тема 2.1: Генерация Compose компонентов из описания (6 часов)

#### Основы промптов для UI-генерации

**Структура эффективного промпта:**
```
Контекст: [тип экрана, цель пользователя]
Компоненты: [какие элементы нужны - Button, TextField, Card и т.д.]
Стиль: [Material 3, кастомная тема, цвета]
Поведение: [клики, валидация, навигация]
Ограничения: [доступность, темная тема, локализация]

Пример:
"Создай экран регистрации пользователя с Material 3. 
Нужны поля: email, пароль, подтверждение пароля.
Валидация: email regex, мин длина пароля 8 символов.
Добавь кнопку 'Войти' и ссылку 'Уже есть аккаунт'.
Поддержка темной темы."
```

**Few-shot prompting для UI:**
```kotlin
// Пример 1: Простая кнопка
@Composable
fun PrimaryButton(
    text: String,
    onClick: () -> Unit
) {
    Button(onClick = onClick) {
        Text(text)
    }
}

// Пример 2: Форма с валидацией
@Composable
fun LoginForm(
    onLogin: (String, String) -> Unit
) {
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    
    Column {
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("Email") }
        )
        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("Password") },
            visualTransformation = PasswordVisualTransformation()
        )
        Button(onClick = { onLogin(email, password) }) {
            Text("Войти")
        }
    }
}

// Теперь сгенерируй экран регистрации на основе этих паттернов...
```

#### Практика: Генерация компонентов для EcoTrack

**Задание 2.1.1:** Создать экран Dashboard с AI (30 минут)

**Промпт для AI:**
```
Создай Composable DashboardScreen для приложения EcoTrack (трекер экологических привычек).

Требования:
1. Header с приветствием пользователя и аватаром
2. Карточки статистики (3 карточки в ряд):
   - Углеродный след за месяц (кг CO2)
   - Экономия денег ($)
   - Посаженные деревья (количество)
3. Список последних активностей (LazyColumn с 5 элементами)
4. FAB для добавления новой активности

Используй Material 3 компоненты, поддерживай темную тему.
Добавь анимации при появлении элементов (FadeIn, SlideIn).
```

**Ожидаемый результат:**
```kotlin
@Composable
fun DashboardScreen(
    user: User,
    stats: EcoStats,
    activities: List<EcoActivity>,
    onAddActivity: () -> Unit,
    modifier: Modifier = Modifier
) {
    Scaffold(
        floatingActionButton = {
            FloatingActionButton(onClick = onAddActivity) {
                Icon(Icons.Default.Add, contentDescription = "Добавить")
            }
        }
    ) { padding ->
        Column(modifier = modifier.padding(padding)) {
            // Header с анимацией
            AnimatedVisibility(visible = true, enter = FadeIn()) {
                UserHeader(user = user)
            }
            
            // Карточки статистики
            StatsCardsRow(stats = stats)
            
            // Список активностей
            ActivitiesList(activities = activities)
        }
    }
}

@Composable
fun StatsCardsRow(stats: EcoStats) {
    Row(horizontalArrangement = Arrangement.spacedBy(16.dp)) {
        StatCard(
            title = "CO2",
            value = "${stats.carbonFootprint} кг",
            icon = Icons.Default.LocalAir,
            color = Color.Green
        )
        StatCard(
            title = "Экономия",
            value = "$${stats.moneySaved}",
            icon = Icons.Default.AttachMoney,
            color = Color.Blue
        )
        StatCard(
            title = "Деревья",
            value = stats.treesPlanted.toString(),
            icon = Icons.Default.Park,
            color = Color.Green
        )
    }
}
```

---

#### Практика: Генерация сложных компонентов (2 часа)

**Задание 2.1.2:** Создать экран AddActivity с валидацией

**Промпт для AI:**
```
Создай экран добавления экологической активности с:

1. Выпадающий список типа активности (Велосипед, Общественный транспорт, Сортировка мусора и т.д.)
2. Поле ввода дистанции (км) или веса (кг) в зависимости от типа
3. Дата и время активности
4. Фото-прикрепление (опционально)
5. Кнопка "Сохранить" с валидацией

Валидация:
- Тип активности обязателен
- Дистанция/вес > 0
- Дата не в будущем

Покажи ошибки под полями, блокируй кнопку до успешной валидации.
Используй Material 3 ExposedDropdownMenuBox и OutlinedTextField.
```

**Ожидаемый результат:**
```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AddActivityScreen(
    onSave: (EcoActivity) -> Unit,
    modifier: Modifier = Modifier
) {
    var activityType by remember { mutableStateOf<ActivityType?>(null) }
    var distance by remember { mutableStateOf("") }
    var date by remember { mutableStateOf(LocalDateTime.now()) }
    
    val errors = remember(activityType, distance) {
        buildList {
            if (activityType == null) add("Выберите тип активности")
            if (distance.isBlank()) add("Укажите дистанцию или вес")
            if (distance.toDoubleOrNull() == null) add("Некорректное значение")
        }
    }
    
    val isFormValid = errors.isEmpty()
    
    Column(modifier = modifier.padding(16.dp)) {
        // Выпадающий список
        ExposedDropdownMenuBox(
            expanded = activityType != null,
            onExpandedChange = { /* toggle */ }
        ) {
            OutlinedTextField(
                value = activityType?.name ?: "",
                onValueChange = {},
                readOnly = true,
                label = { Text("Тип активности") },
                trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded = activityType != null) },
                modifier = Modifier.menuAnchor()
            )
            
            ExposedDropdownMenu(expanded = activityType != null, onDismissRequest = { /* close */ }) {
                ActivityType.entries.forEach { type ->
                    DropdownMenuItem(
                        text = { Text(type.displayName) },
                        onClick = { 
                            activityType = type
                        }
                    )
                }
            }
        }
        
        // Поле дистанции/веса
        OutlinedTextField(
            value = distance,
            onValueChange = { distance = it },
            label = { Text("Дистанция (км)") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Decimal),
            supportingText = if (errors.isNotEmpty()) {
                { Text(errors.joinToString("\n"), color = MaterialTheme.colorScheme.error) }
            } else null
        )
        
        // Дата и время
        DateTimePicker(
            selectedDate = date,
            onDateChange = { date = it }
        )
        
        Spacer(modifier = Modifier.height(24.dp))
        
        // Кнопка сохранения
        Button(
            onClick = { 
                if (isFormValid) {
                    onSave(EcoActivity(
                        type = activityType!!,
                        distance = distance.toDouble(),
                        date = date
                    ))
                }
            },
            enabled = isFormValid,
            colors = ButtonDefaults.buttonColors(
                disabledContainerColor = MaterialTheme.colorScheme.primary.copy(alpha = 0.3f)
            )
        ) {
            Text("Сохранить")
        }
    }
}
```

---

### Тема 2.2: Создание дизайн-системы через AI (4 часа)

#### Генерация темы Material 3 из описания

**Промпт для AI:**
```
Создай Material 3 тему для приложения EcoTrack с экологической стилистикой.

Основные цвета:
- Primary: #2E7D32 (темно-зеленый)
- Secondary: #66BB6A (светло-зеленый)  
- Tertiary: #FFA000 (оранжевый для акцентов)
- Background: #F1F8E9 (светло-зеленый фон)

Типографика:
- Headline: Roboto Bold 24sp
- Title: Roboto Medium 18sp  
- Body: Roboto Regular 16sp

Spacing:
- xs = 4dp, sm = 8dp, md = 16dp, lg = 24dp, xl = 32dp

Создай файл Theme.kt с:
1. ColorScheme для светлой и темной темы
2. Typography с fontFamily = Roboto
3. MaterialTheme wrapper
4. Extension properties для spacing (Modifier.paddingXs(), etc.)

Поддержка темной темы:
- Primary: #81C784 (светлее)
- Background: #1B5E20 (темно-зеленый)
```

**Ожидаемый результат:**
```kotlin
// Theme.kt
import androidx.compose.foundation.text.appendString
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.unit.dp

object EcoTrackColors {
    // Светлая тема
    val PrimaryLight = Color(0xFF2E7D32)
    val SecondaryLight = Color(0xFF66BB6A)
    val TertiaryLight = Color(0xFFFFA000)
    val BackgroundLight = Color(0xFFF1F8E9)
    
    // Темная тема
    val PrimaryDark = Color(0xFF81C784)
    val SecondaryDark = Color(0xFF388E3C)
    val TertiaryDark = Color(0xFFFFB74D)
    val BackgroundDark = Color(0xFF1B5E20)
}

object EcoTrackSpacing {
    val xs = 4.dp
    val sm = 8.dp
    val md = 16.dp
    val lg = 24.dp
    val xl = 32.dp
}

val LightColorScheme = lightColorScheme(
    primary = EcoTrackColors.PrimaryLight,
    secondary = EcoTrackColors.SecondaryLight,
    tertiary = EcoTrackColors.TertiaryLight,
    background = EcoTrackColors.BackgroundLight,
    surface = Color.White,
    error = Color(0xFFB00020)
)

val DarkColorScheme = darkColorScheme(
    primary = EcoTrackColors.PrimaryDark,
    secondary = EcoTrackColors.SecondaryDark,
    tertiary = EcoTrackColors.TertiaryDark,
    background = EcoTrackColors.BackgroundDark,
    surface = Color(0xFF1B5E20),
    error = Color(0xFFFFB4AB)
)

val EcoTrackTypography = Typography(
    headlineLarge = TextStyle(
        fontFamily = FontFamily.Default, // Roboto по умолчанию на Android
        fontSize = 24.sp,
        fontWeight = FontWeight.Bold
    ),
    titleLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontSize = 18.sp,
        fontWeight = FontWeight.Medium
    ),
    bodyLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontSize = 16.sp,
        fontWeight = FontWeight.Normal
    )
)

@Composable
fun EcoTrackTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = EcoTrackTypography,
        content = content
    )
}

// Extension properties для spacing
fun Modifier.paddingXs() = this.padding(EcoTrackSpacing.xs)
fun Modifier.paddingSm() = this.padding(EcoTrackSpacing.sm)
fun Modifier.paddingMd() = this.padding(EcoTrackSpacing.md)
fun Modifier.paddingLg() = this.padding(EcoTrackSpacing.lg)
fun Modifier.paddingXl() = this.padding(EcoTrackSpacing.xl)

fun Modifier.paddingHorizontalSm() = this.padding(horizontal = EcoTrackSpacing.sm)
fun Modifier.paddingVerticalMd() = this.padding(vertical = EcoTrackSpacing.md)
```

---

#### Генерация компонентов дизайн-системы (2 часа)

**Задание 2.2.1:** Создать кастомные компоненты из Figma макета

**Промпт для AI:**
```
На основе дизайн-системы EcoTrack создай следующие кастомные компоненты:

1. EcoCard - карточка с тенью, закругленными углами (16dp), 
   содержит: иконку, заголовок, значение, тренд (стрелка вверх/вниз)

2. ActivityChip - чип для отображения типа активности с иконкой
   Поддерживает selected state

3. ProgressRing - круговой прогресс-бар для отображения выполнения целей
   С анимацией заполнения

4. StatCard - карточка статистики с иконкой, заголовком, значением
   Поддерживает hover state (для desktop)

Используй Material 3 компоненты как основу, добавь кастомную стилизацию.
Все компоненты должны поддерживать темную тему через MaterialTheme.colorScheme.
```

**Ожидаемый результат:**
```kotlin
@Composable
fun EcoCard(
    icon: ImageVector,
    title: String,
    value: String,
    trend: Trend? = null, // Up, Down, Neutral
    modifier: Modifier = Modifier
) {
    Card(
        modifier = modifier
            .paddingMd()
            .shadow(
                elevation = 4.dp,
                shape = RoundedCornerShape(16.dp),
                clipToShape = true
            ),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Column(
            modifier = Modifier
                .paddingMd()
                .fillMaxWidth(),
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            // Иконка в круге
            Surface(
                shape = CircleShape,
                color = MaterialTheme.colorScheme.primary.copy(alpha = 0.1f),
                modifier = Modifier.size(48.dp)
            ) {
                Icon(
                    icon,
                    contentDescription = null,
                    tint = MaterialTheme.colorScheme.primary,
                    modifier = Modifier.padding(12.dp)
                )
            }
            
            Spacer(modifier = Modifier.height(EcoTrackSpacing.sm))
            
            // Заголовок
            Text(
                text = title,
                style = MaterialTheme.typography.titleMedium,
                color = MaterialTheme.colorScheme.onSurface.copy(alpha = 0.6f)
            )
            
            Spacer(modifier = Modifier.height(4.dp))
            
            // Значение
            Text(
                text = value,
                style = MaterialTheme.typography.headlineMedium,
                color = MaterialTheme.colorScheme.onSurface
            )
            
            // Тренд (если есть)
            trend?.let { trend ->
                Spacer(modifier = Modifier.height(4.dp))
                Row(
                    horizontalArrangement = Arrangement.Center,
                    verticalAlignment = Alignment.Bottom
                ) {
                    Icon(
                        imageVector = when (trend) {
                            Trend.Up -> Icons.Default.ArrowUpward
                            Trend.Down -> Icons.Default.ArrowDownward
                            else -> Icons.Default.Remove
                        },
                        contentDescription = null,
                        tint = when (trend) {
                            Trend.Up -> Color.Green
                            Trend.Down -> Color.Red
                            else -> MaterialTheme.colorScheme.onSurface.copy(alpha = 0.6f)
                        },
                        modifier = Modifier.size(16.dp)
                    )
                }
            }
        }
    }
}

@Composable
fun ActivityChip(
    type: ActivityType,
    selected: Boolean = false,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    FilterChip(
        selected = selected,
        onClick = onClick,
        modifier = modifier,
        label = { Text(type.displayName) },
        leadingIcon = if (selected) {
            { Icon(type.icon, contentDescription = null) }
        } else null,
        colors = FilterChipDefaults.filterChipColors(
            containerColor = if (selected) {
                MaterialTheme.colorScheme.primary.copy(alpha = 0.1f)
            } else {
                MaterialTheme.colorScheme.surfaceVariant
            },
            labelColor = if (selected) {
                MaterialTheme.colorScheme.primary
            } else {
                MaterialTheme.colorScheme.onSurfaceVariant
            }
        )
    )
}

@Composable
fun ProgressRing(
    progress: Float, // 0.0 to 1.0
    size: Dp = 120.dp,
    modifier: Modifier = Modifier
) {
    val ringColor = MaterialTheme.colorScheme.primary
    val backgroundColor = ringColor.copy(alpha = 0.2f)
    
    Canvas(
        modifier = modifier.size(size)
    ) {
        val center = size.value / 2
        val radius = center - 8f // отступ для толщины кольца
        
        // Фоновое кольцо
        drawArc(
            color = backgroundColor,
            startAngle = -90f,
            sweepAngle = 360f,
            useCenter = false,
            topLeft = Offset(center - radius, center - radius),
            size = Size(radius * 2, radius * 2)
        )
        
        // Прогресс кольцо с анимацией
        val animatedProgress = rememberAnimatedFloatState(progress)
        drawArc(
            color = ringColor,
            startAngle = -90f,
            sweepAngle = 360f * animatedProgress.value,
            useCenter = false,
            topLeft = Offset(center - radius, center - radius),
            size = Size(radius * 2, radius * 2)
        )
    }
}

enum class Trend {
    Up, Down, Neutral
}
```

---

### Тема 2.3: AI как фича в приложении (4 часа)

#### Интеграция LLM API для пользовательских фич

**Промпт для AI:**
```
Создай чат-бота для EcoTrack, который помогает пользователям:

1. Рекомендует экологические привычки на основе их статистики
2. Отвечает на вопросы о углеродном следе разных действий
3. Помогает ставить цели по снижению CO2

Требования:
- REST API клиент для вызова LLM (OpenAI/Claude)
- Чат интерфейс с историей сообщений
- System prompt для настройки поведения бота
- Кэширование ответов для повторяющихся вопросов

Создай:
1. ChatScreen с LazyColumn сообщений
2. EcoBotService для вызова LLM API
3. System prompt с контекстом о экологии и углеродном следе
```

**Ожидаемый результат:**
```kotlin
// EcoBotService.kt
class EcoBotService(
    private val apiKey: String,
    private val cache: MutableMap<String, String> = mutableMapOf()
) {
    companion object {
        const val SYSTEM_PROMPT = """
            Ты - экологический помощник EcoBot в приложении EcoTrack.
            
            Твоя задача:
            1. Помогать пользователям снижать углеродный след
            2. Рекомендовать экологические привычки (велосипед, сортировка мусора, энергосбережение)
            3. Объяснять влияние разных действий на экологию
            
            Тон: дружелюбный, мотивирующий, но фактологически точный.
            Не выдумывай статистику - если не знаешь, скажи "мне нужно проверить данные".
            
            Примеры ответов:
            - "Езда на велосипеде вместо машины экономит ~2.3 кг CO2 на 10 км"
            - "Сортировка мусора может снизить ваш углеродный след на 15%"
            - "Отказ от одноразовых пластиковых бутылок экономит ~150 кг CO2 в год"
        """.trimIndent()
    }
    
    suspend fun chat(message: String, history: List<ChatMessage>): ChatResponse {
        // Проверка кэша для повторяющихся вопросов
        val cacheKey = normalizeMessage(message)
        if (cache.containsKey(cacheKey)) {
            return ChatResponse(
                content = cache[cacheKey]!!,
                isCached = true
            )
        }
        
        // Формирование запроса к LLM API
        val messages = buildList {
            add(ChatMessage(role = "system", content = SYSTEM_PROMPT))
            addAll(history)
            add(ChatMessage(role = "user", content = message))
        }
        
        val response = callLLMAPI(messages)
        
        // Кэширование ответа
        cache[cacheKey] = response.content
        
        return ChatResponse(
            content = response.content,
            isCached = false
        )
    }
    
    private suspend fun callLLMAPI(messages: List<ChatMessage>): ChatResponse {
        // Реализация вызова OpenAI/Claude API через Ktor Client
        // ...
    }
    
    private fun normalizeMessage(message: String): String {
        return message.lowercase().replace(Regex("\\s+"), " ").trim()
    }
}

// ChatScreen.kt
@Composable
fun EcoBotChatScreen(
    botService: EcoBotService,
    modifier: Modifier = Modifier
) {
    var userMessage by remember { mutableStateOf("") }
    val messages = remember { mutableStateListOf<ChatMessage>() }
    
    LaunchedEffect(Unit) {
        // Приветственное сообщение от бота
        messages.add(ChatMessage(
            role = "assistant",
            content = "Привет! Я EcoBot - твой экологический помощник. Чем могу помочь в снижении углеродного следа? 🌱"
        ))
    }
    
    Column(modifier = modifier.fillMaxSize()) {
        // Список сообщений
        LazyColumn(
            modifier = Modifier.weight(1f),
            contentPadding = PaddingValues(16.dp)
        ) {
            items(messages) { message ->
                ChatBubble(
                    message = message,
                    isUser = message.role == "user"
                )
            }
        }
        
        // Поле ввода
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp),
            horizontalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            OutlinedTextField(
                value = userMessage,
                onValueChange = { userMessage = it },
                modifier = Modifier.weight(1f),
                placeholder = { Text("Спроси о экологических привычках...") },
                singleLine = true
            )
            
            IconButton(
                onClick = {
                    if (userMessage.isNotBlank()) {
                        messages.add(ChatMessage(role = "user", content = userMessage))
                        
                        val currentMessage = userMessage
                        userMessage = ""
                        
                        // Асинхронный запрос к боту
                        launch {
                            val response = botService.chat(currentMessage, messages.filter { it.role != "user" })
                            messages.add(ChatMessage(role = "assistant", content = response.content))
                        }
                    }
                },
                enabled = userMessage.isNotBlank()
            ) {
                Icon(Icons.Default.Send, contentDescription = "Отправить")
            }
        }
    }
}

@Composable
fun ChatBubble(
    message: ChatMessage,
    isUser: Boolean,
    modifier: Modifier = Modifier
) {
    val backgroundColor = if (isUser) {
        MaterialTheme.colorScheme.primary
    } else {
        MaterialTheme.colorScheme.surfaceVariant
    }
    
    val textColor = if (isUser) {
        MaterialTheme.colorScheme.onPrimary
    } else {
        MaterialTheme.colorScheme.onSurfaceVariant
    }
    
    Bubble(
        modifier = modifier
            .paddingVerticalSm()
            .align(if (isUser) Alignment.End else Alignment.Start),
        containerColor = backgroundColor,
        contentColor = textColor
    ) {
        Text(text = message.content)
        
        if (!isUser && message.isCached) {
            Text(
                text = " (из кэша)",
                style = MaterialTheme.typography.labelSmall,
                color = textColor.copy(alpha = 0.6f)
            )
        }
    }
}

data class ChatMessage(
    val role: String, // "user" или "assistant"
    val content: String,
    val isCached: Boolean = false
)

data class ChatResponse(
    val content: String,
    val isCached: Boolean = false
)
```

---

### Тема 2.4: Оптимизация производительности UI (2 часа)

#### AI для анализа и оптимизации Compose кода

**Промпт для AI:**
```
Проанализируй следующий Composable и предложи оптимизации:

[вставь код компонента]

Обрати внимание на:
1. Непотребные recomposition (использование remember, derivedStateOf)
2. Неэффективные списки (LazyColumn вместо Column для больших данных)
3. Излишние state-объекты (использование State vs mutableStateOf)
4. Оптимизация анимаций (AnimatedVisibility вместо полной перерисовки)

Предложи оптимизированную версию с комментариями что и почему изменено.
```

**Пример оптимизации:**

**До (неоптимально):**
```kotlin
@Composable
fun UserList(users: List<User>) {
    Column {
        users.forEach { user ->
            UserCard(user = user) // Recomposition при любом изменении в users
        }
    }
}
```

**После (оптимизировано):**
```kotlin
@Composable
fun UserList(users: List<User>) {
    LazyColumn {
        items(
            items = users,
            key = { user -> user.id } // Ключ для эффективного обновления
        ) { user ->
            UserCard(user = user) // Recomposition только для измененного элемента
        }
    }
}

@Composable
fun UserCard(user: User) {
    // Использование derivedStateOf для вычисляемых значений
    val isOnline = remember(user) {
        derivedStateOf { user.lastSeen < 5.minutes.ago }
    }
    
    Column {
        Row {
            Avatar(user = user)
            Spacer(modifier = Modifier.width(8.dp))
            Column {
                Text(text = user.name, style = MaterialTheme.typography.titleMedium)
                Text(
                    text = if (isOnline.value) "Онлайн" else "Был ${user.lastSeen}",
                    style = MaterialTheme.typography.bodySmall,
                    color = if (isOnline.value) Color.Green else Color.Gray
                )
            }
        }
    }
}
```

---

## 📝 Практические задания модуля

### Задание 2.1: Создать 5 экранов EcoTrack с AI (4 часа)

**Экраны:**
1. Dashboard - статистика и последние активности
2. AddActivity - форма добавления с валидацией
3. ActivityDetail - детали активности с графиком
4. Goals - цели и прогресс-бары
5. Profile - настройки пользователя

**Критерии:**
- ✓ Все экраны используют Material 3 компоненты
- ✓ Поддержка темной темы через EcoTrackTheme
- ✓ Валидация форм с отображением ошибок
- ✓ Анимации переходов между экранами

---

### Задание 2.2: Создать дизайн-систему из Figma макета (3 часа)

**Входные данные:**
- Figma файл с 10+ экранами приложения
- Color palette, typography, spacing guidelines

**Результат:**
- Theme.kt с ColorScheme и Typography
- 10+ кастомных компонентов (EcoCard, ActivityChip, ProgressRing и т.д.)
- Extension properties для spacing

**Критерии:**
- ✓ Все компоненты поддерживают темную тему
- ✓ Консистентный дизайн через всю систему
- ✓ Документация по использованию компонентов

---

### Задание 2.3: Интегрировать EcoBot чат-бота (3 часа)

**Требования:**
- REST API клиент для OpenAI/Claude
- Чат интерфейс с историей сообщений
- System prompt с экологическим контекстом
- Кэширование ответов

**Критерии:**
- ✓ Бот отвечает на вопросы о экологии и углеродном следе
- ✓ Кэширование работает для повторяющихся вопросов
- ✓ UI поддерживает длинные сообщения и скроллинг

---

## 📚 Дополнительные материалы

### Документация:
- [Jetpack Compose Documentation](https://developer.android.com/jetpack/compose)
- [Material 3 Components](https://m3.material.io/components)
- [Compose Performance Best Practices](https://developer.android.com/jetpack/compose/performance)

### Инструменты:
- [Figma to Compose plugins](https://www.figma.com/community/plugin/1468573022978173f2a5/Figma-to-Compose)
- [Composable Preview](https://developer.android.com/jetpack/compose/preview)

### Видео:
- "Building Beautiful UIs with Jetpack Compose" (Android Developers, 45 мин)
- "Material 3 Theming Deep Dive" (YouTube, 30 мин)

---

## 🚀 Следующий шаг

Переходите к [Модулю 3](../Module_03_AI_Architecture/content.md): AI в архитектуре и бизнес-логике

**Время до следующего модуля:** 2 недели  
**Рекомендуемая практика:** Ежедневно генерировать минимум 1 Composable компонент с AI

---

**Удачи в создании красивых UI с помощью ИИ! 🎨🤖**

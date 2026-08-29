# 📘 Модуль 2: Advanced UI Patterns

**В этом модуле вы освоите продвинутые паттерны UI для KMP: Compose Multiplatform advanced features, SwiftUI интеграция, custom composables и state management.**

**Цели модуля:**
1. Создать сложные UI с Compose Multiplatform
2. Интегрировать KMP с SwiftUI для iOS
3. Реализовать custom composables и анимации
4. Настроить state management (Unidirectional Data Flow)

**Время выполнения:** ~25 часов.

---

## 1. Compose Multiplatform Advanced Patterns

### Custom Layouts:

```kotlin
// commonMain - Custom layout для Masonry grid
@Composable
fun MasonryGrid(
    items: List<Item>,
    columns: Int = 2,
    itemContent: @Composable (Item) -> Unit
) {
    LazyColumn(
        contentPadding = PaddingValues(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(items.chunked(columns)) { rowItems ->
            Row(
                horizontalArrangement = Arrangement.spacedBy(8.dp),
                modifier = FillMaxWidth()
            ) {
                rowItems.forEach { item ->
                    Box(
                        modifier = Weight(1f).fillMaxHeight()
                    ) {
                        itemContent(item)
                    }
                }
            }
        }
    }
}

// Advanced: Custom Layout with MeasurePolicy
@Composable
fun StaggeredGrid(
    items: List<Item>,
    modifier: Modifier = Modifier,
    itemContent: @Composable (Item) -> Unit
) {
    Layout(
        content = {
            items.forEach { item ->
                Box(key = item.id) {
                    itemContent(item)
                }
            }
        },
        modifier = modifier
    ) { measurables, constraints ->
        val placeables = measurables.map { measurable ->
            measurable.measure(constraints)
        }
        
        // Custom layout logic (staggered grid algorithm)
        val columnWidth = constraints.maxWidth / 2
        
        var leftY = 0
        var rightY = 0
        
        placeables.forEachIndexed { index, placeable ->
            val x = if (leftY < rightY) 0 else columnWidth
            val y = if (x == 0) {
                leftY += placeable.height
                leftY - placeable.height
            } else {
                rightY += placeable.height
                rightY - placeable.height
            }
            
            placeable.place(x, y)
        }
        
        layout(
            constraints.maxWidth,
            max(leftY, rightY)
        )
    }
}
```

### Advanced Animations:

```kotlin
@Composable
fun AnimatedListItem(
    item: Item,
    onClick: () -> Unit
) {
    var isExpanded by remember { mutableStateOf(false) }
    
    Card(
        onClick = { isExpanded = !isExpanded },
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Column {
            // Header - always visible
            Row(
                modifier = PaddingValues(16.dp),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text(text = item.title, style = MaterialTheme.typography.h6)
                
                AnimatedVisibility(
                    visible = isExpanded,
                    transitionSpec = { fadeIn() + scaleIn(initialScale = 0.5f) }
                ) {
                    IconButton(onClick = onClick) {
                        Icon(Icons.Default.Check, contentDescription = "Select")
                    }
                }
            }
            
            // Expanded content - animated
            AnimatedVisibility(
                visible = isExpanded,
                transitionSpec = { 
                    fadeIn() + expandVertically() 
                        .usingSpring(Spring.StiffnessLow)
                }
            ) {
                PaddingValues(vertical = 8.dp, horizontal = 16.dp).let { padding ->
                    Column(modifier = Modifier.padding(padding)) {
                        Text(text = item.description)
                        
                        // Animated list of tags
                        LazyRow(
                            modifier = Modifier.padding(top = 8.dp),
                            horizontalArrangement = Arrangement.spacedBy(8.dp)
                        ) {
                            items(item.tags) { tag ->
                                AnimatedContent(tag) { animatedTag ->
                                    Chip(text = animatedTag)
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

// Shared animation specs
object AnimationSpecs {
    val Fast = tween(durationMillis = 200, easing = FastOutSlowInEasing)
    val Normal = tween(durationMillis = 300, easing = FastOutSlowInEasing)
    val Slow = tween(durationMillis = 500, easing = FastOutSlowInEasing)
    
    val Spring = spring(stiffness = Spring.StiffnessLow)
}
```

### State Management с Unidirectional Data Flow:

```kotlin
// UI State (immutable)
data class ScreenState(
    val items: List<Item> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val searchQuery: String = ""
)

// UI Events (user interactions)
sealed class ScreenEvent {
    object Refresh : ScreenEvent()
    data class Search(val query: String) : ScreenEvent()
    data class SelectItem(val itemId: String) : ScreenEvent()
}

// ViewModel с MVI pattern
class ScreenViewModel(
    private val repository: ItemRepository
) : ViewModel() {
    
    // StateFlow для UI state
    private val _state = MutableStateFlow(ScreenState())
    val state: StateFlow<ScreenState> = _state.asStateFlow()
    
    // Channel для events
    private val _events = MutableSharedFlow<ScreenEvent>()
    val events: SharedFlow<ScreenEvent> = _events
    
    init {
        // Запускаем coroutine для обработки events
        viewModelScope.launch {
            _events.collect { event ->
                handleEvent(event)
            }
        }
        
        // Initial load
        loadItems()
    }
    
    private suspend fun handleEvent(event: ScreenEvent) {
        when (event) {
            is ScreenEvent.Refresh -> loadItems()
            is ScreenEvent.Search -> searchItems(event.query)
            is ScreenEvent.SelectItem -> selectItem(event.itemId)
        }
    }
    
    private suspend fun loadItems() {
        _state.update { it.copy(isLoading = true) }
        
        try {
            val items = repository.getItems()
            _state.update { it.copy(items = items, isLoading = false, error = null) }
        } catch (e: Exception) {
            _state.update { it.copy(isLoading = false, error = e.message) }
        }
    }
    
    private suspend fun searchItems(query: String) {
        _state.update { it.copy(searchQuery = query, isLoading = true) }
        
        try {
            val items = repository.searchItems(query)
            _state.update { it.copy(items = items, isLoading = false) }
        } catch (e: Exception) {
            _state.update { it.copy(isLoading = false, error = e.message) }
        }
    }
    
    private suspend fun selectItem(itemId: String) {
        // Emit event для parent screen
        _events.emit(ScreenEvent.SelectItem(itemId))
    }
}

// Composable с MVI
@Composable
fun Screen(
    viewModel: ScreenViewModel = hiltViewModel()
) {
    val state by viewModel.state.collectAsState()
    
    ScreenContent(
        state = state,
        onEvent = { event -> 
            viewModel._events.tryEmit(event) 
        }
    )
}

@Composable
private fun ScreenContent(
    state: ScreenState,
    onEvent: (ScreenEvent) -> Unit
) {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Items") },
                actions = {
                    IconButton(onClick = { onEvent(ScreenEvent.Refresh) }) {
                        Icon(
                            imageVector = if (state.isLoading) 
                                Icons.Default.Autorenew else Icons.Default.Refresh,
                            contentDescription = "Refresh",
                            modifier = Modifier.then(
                                if (state.isLoading) 
                                    Modifier.rotateAnimatable(Animatable(0f)).rotateAnimation()
                                else Modifier
                            )
                        )
                    }
                }
            )
        }
    ) { padding ->
        Box(modifier = Modifier.padding(padding)) {
            when {
                state.isLoading && state.items.isEmpty() -> {
                    CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
                }
                
                state.error != null -> {
                    Column(modifier = Modifier.align(Alignment.Center)) {
                        Text("Error: ${state.error}")
                        Button(onClick = { onEvent(ScreenEvent.Refresh) }) {
                            Text("Retry")
                        }
                    }
                }
                
                state.items.isEmpty() -> {
                    Text("No items found", modifier = Modifier.align(Alignment.Center))
                }
                
                else -> {
                    LazyColumn {
                        items(state.items) { item ->
                            AnimatedListItem(
                                item = item,
                                onClick = { onEvent(ScreenEvent.SelectItem(item.id)) }
                            )
                        }
                    }
                }
            }
        }
    }
}

// Extension для анимации вращения
fun Modifier.rotateAnimation(): Modifier {
    val infiniteTransition = rememberInfiniteTransition()
    val rotation by infiniteTransition.animateFloat(
        initialValue = 0f,
        targetValue = 360f,
        animationSpec = infiniteRepeatable(
            animation = tween(durationMillis = 1000),
            repeatMode = RepeatMode.Restart
        )
    )
    
    return this.rotate(rotation)
}
```

---

## 2. SwiftUI Integration для iOS

### Expect/Actual для UI компонентов:

```kotlin
// commonMain - UI interface
expect fun createSettingsScreen(
    onSettingChanged: (String, String) -> Unit
): PlatformView

// iosMain - SwiftUI implementation
actual fun createSettingsScreen(
    onSettingChanged: (String, String) -> Unit
): PlatformView {
    val settingsViewController = SettingsViewController(onSettingChanged)
    return IOSPlatformView(settingsViewController)
}

// SwiftUI ViewController
class SettingsViewController(
    private val onSettingChanged: (String, String) -> Unit
) : UIViewController() {
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let hostingController = UIHostingController(
            rootView: SettingsView(onSettingChanged: onSettingChanged)
        )
        
        addChild(hostingController)
        view.addSubview(hostingController.view)
        
        hostingController.view.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            hostingController.view.topAnchor.constraint(equalTo: view.topAnchor),
            hostingController.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            hostingController.view.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            hostingController.view.bottomAnchor.constraint(equalTo: view.bottomAnchor)
        ])
        
        hostingController.didMove(toParent: self)
    }
}

// SwiftUI View
struct SettingsView: View {
    let onSettingChanged: (String, String) -> Void
    
    @State private var theme = "light"
    @State private var language = "en"
    
    var body: some View {
        List {
            Section(header: Text("Appearance")) {
                Picker("Theme", selection: $theme) {
                    Text("Light").tag("light")
                    Text("Dark").tag("dark")
                }
                
                .onChange(of: theme) { newValue in
                    onSettingChanged("theme", newValue)
                }
            }
            
            Section(header: Text("Language")) {
                Picker("Language", selection: $language) {
                    Text("English").tag("en")
                    Text("Русский").tag("ru")
                }
                
                .onChange(of: language) { newValue in
                    onSettingChanged("language", newValue)
                }
            }
        }
        .navigationTitle("Settings")
    }
}
```

---

## 📝 Домашнее задание (Модуль 2)

### Задача: Создание advanced UI для SkillSync

**Требования:**
1. Реализуйте Masonry Grid layout для отображения навыков
2. Добавьте анимации (expand/collapse, transitions)
3. Настройте MVI state management для всех экранов
4. Интегрируйте SwiftUI для iOS-specific screens

**Критерии сдачи:**
- ✅ Custom layouts работают на Android и iOS
- ✅ Анимации плавные (60 FPS)
- ✅ State management через MVI pattern
- ✅ SwiftUI screens для iOS

---

**Следующий модуль:** В Module_03 мы изучим advanced database patterns с SQLDelight и Realm.

Удачи! 🚀

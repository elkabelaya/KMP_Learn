# 📝 Самопроверка: Модуль 4 — Advanced UI Patterns

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 11+ из 14 без подглядывания.

---

## Вопрос 1. Чем отличаются подходы `MasonryGrid` и `StaggeredGrid` из модуля?

- **A)** `MasonryGrid` собирается из встроенных `LazyColumn` + `chunked()` + `Row`/`Weight`, а `StaggeredGrid` использует кастомный `Layout` с `MeasurePolicy` и ручной `measure`/`place`
- **B)** Оба используют `MeasurePolicy`, просто у `StaggeredGrid` больше колонок
- **C)** `MasonryGrid` работает только на Android, `StaggeredGrid` — только на iOS
- **D)** Разницы нет, это два имени одного и того же composable

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В content.md это два разных уровня кастомизации:

- `MasonryGrid` — «простой» путь: `items(items.chunked(columns))` разбивает список на строки, дальше `Row` с `horizontalArrangement = spacedBy(8.dp)` и `Box(Weight(1f))`. Ячейки в строке получают равную ширину, но **строки** здесь обычные — это сетка «по строкам».
- `StaggeredGrid` — настоящий кастомный layout: `Layout(content = ...) { measurables, constraints -> ... }`. Внутри сам проект `measurables.map { it.measure(constraints) }`, сам считает координаты и сам вызывает `place(x, y)` — элементы «каскадом» заполняют более короткую колонку.

**Почему не другие:**
- B — `MasonryGrid` не использует `MeasurePolicy`, он состоит из готовых composable
- C — оба находятся в `commonMain` и работают на всех платформах Compose Multiplatform
- D — это принципиально разные реализации (декларативная композиция vs ручной measure/layout)

</details>

---

## Вопрос 2. Фрагмент из `StaggeredGrid`:

```kotlin
val placeables = measurables.map { measurable ->
    measurable.measure(constraints)
}

val columnWidth = constraints.maxWidth / 2
var leftY = 0
var rightY = 0
// ... размещение элементов, обновляющее leftY / rightY ...

layout(
    constraints.maxWidth,
    max(leftY, rightY)
)
```

Какую **высоту** будет иметь итоговый layout?

- **A)** `constraints.maxHeight` — всегда максимальную доступную
- **B)** `max(leftY, rightY)` — высоту самой высокой из двух колонок
- **C)** `leftY + rightY` — сумму высот обеих колонок
- **D)** `0`, пока не будет вызван `place`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В `MeasurePolicy` последний вызов `layout(width, height)` задаёт итоговый размер контейнера. Здесь ширина — `constraints.maxWidth` (на всю доступную), а высота — `max(leftY, rightY)`: после размещения всех элементов в `leftY` и `rightY` накоплены высоты левой и правой колонок. Контейнер должен быть ровно таким, чтобы вместить **самую длинную** колонку — не больше и не меньше.

**Почему не другие:**
- A — `constraints.maxHeight` здесь не используется; кастомный layout сам решает, сколько места занять (обычно обжимается по контенту)
- C — сумма завысит высоту вдвое: колонки идут параллельно, а не друг под другом
- D — `place` лишь задаёт позиции уже измеренных `placeables`; размер определяется вызовом `layout(...)`

</details>

---

## Вопрос 3. Фрагмент логики размещения из `StaggeredGrid`:

```kotlin
val columnWidth = constraints.maxWidth / 2

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
```

Какую задачу решает выбор `x` через `if (leftY < rightY)`?

- **A)** Кладет очередной элемент в **более короткую** колонку, балансируя высоты (собственно «staggered»)
- **B)** Всегда кладёт элемент в левую колонку
- **C)** Строит строгое чередование «чётный → левая, нечётный → правая» по индексу
- **D)** Выравнивает элемент по горизонтальному центру

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`leftY < rightY` означает «левая колонка пока короче» → `x = 0` (левая). Иначе `x = columnWidth` (правая). После этого `y` берётся как накопленная высота выбранной колонки **до** увеличения (`leftY += height`, затем `leftY - height` = старое `leftY`). Это и есть алгоритм «вставляй в самую короткую колонку», из-за которого низ получается рваным (staggered), но сбалансированным.

**Почему не другие:**
- B — тогда правая колонка никогда бы не использовалась и условие было бы бессмысленным
- C — индекс `index` здесь вообще не участвует в выборе `x`; решение принимается по текущим высотам
- D — для центра нужен расчёт `(constraints.maxWidth - placeable.width) / 2`, а не две фиксированные колонки

</details>

---

## Вопрос 4. В `AnimatedListItem` состояние раскрытия объявлено так:

```kotlin
var isExpanded by remember { mutableStateOf(false) }
```

Что о нём верно?

- **A)** Значение живёт в Compose-дереве; изменение вызывает рекомпозицию, но состояние теряется при выходе composable со экрана (и при смене ключа/конфигурации), если не использовать `rememberSaveable`
- **B)** Состояние хранится в `ScreenViewModel` и переживает все конфигурационные изменения
- **C)** Одно значение `isExpanded` общее для всех элементов списка
- **D)** `mutableStateOf` не вызывает рекомпозицию — обновление произойдёт только после ручного вызова

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`remember` сохраняет значение в слоте композиции, пока composable остаётся в дереве. `by mutableStateOf(false)` с делегатом `by` делает чтение/запись видимыми для Compose: при `isExpanded = !isExpanded` произойдёт рекомпозиция, и `AnimatedVisibility` переключится. Но `remember` **не** выживает при уничтожении активности/конфигурации — для этого в модуле использовались бы `rememberSaveable` или подъём состояния в ViewModel.

**Почему не другие:**
- B — в коде это локальная переменная composable, а не поле ViewModel
- C — функция `AnimatedListItem` вызывается для каждого item, у каждого вызова свой `remember`
- D — как раз наоборот: чтение `State` внутри composable автоматически подписывает его на изменения

</details>

---

## Вопрос 5. Что делает эта анимация иконки из `AnimatedListItem`?

```kotlin
AnimatedVisibility(
    visible = isExpanded,
    transitionSpec = { fadeIn() + scaleIn(initialScale = 0.5f) }
) {
    IconButton(onClick = onClick) {
        Icon(Icons.Default.Check, contentDescription = "Select")
    }
}
```

- **A)** Иконка появляется с проявлением (fade) и увеличением **с половины размера до полного**
- **B)** Иконка исчезает, сжимаясь до нуля
- **C)** Иконка бесконечно вращается на 360°
- **D)** Иконка выезжает сверху с пружинным эффектом

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`visible = isExpanded` → при раскрытии карточки иконка становится видна. `fadeIn()` отвечает за прозрачность 0 → 1, а `scaleIn(initialScale = 0.5f)` — за масштаб 0.5f → 1f. Обе анимации складываются через `+`, то есть идут одновременно.

**Почему не другие:**
- B — это был бы `fadeOut()` + `scaleOut()` и `visible = !isExpanded`
- C — вращение делает `rotateAnimation()` из этого же модуля (`infiniteRepeatable`), а не `scaleIn`
- D — выезжание сверху — это `slideInVertically()`; здесь его нет

</details>

---

## Вопрос 6. Про `object AnimationSpecs` из модуля:

```kotlin
val Fast = tween(durationMillis = 200, easing = FastOutSlowInEasing)
val Normal = tween(durationMillis = 300, easing = FastOutSlowInEasing)
val Slow = tween(durationMillis = 500, easing = FastOutSlowInEasing)
val Spring = spring(stiffness = Spring.StiffnessLow)
```

Какая длительность у `Fast` и что означает `StiffnessLow` у пружины?

- **A)** 200 мс; низкая жёсткость — мягкая, «более долгая» пружина с плавным затуханием
- **B)** 300 мс; низкая жёсткость — мгновенный переход без анимации
- **C)** 500 мс; жёсткость не влияет на поведение пружины
- **D)** 200 мс; низкая жёсткость — очень резкий и быстрый «щелчок»

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`Fast = 200 мс`, `Normal = 300 мс`, `Slow = 500 мс` — все с `FastOutSlowInEasing` (быстрый старт, плавный финиш). `spring(stiffness = Spring.StiffnessLow)` — пружина с **низкой** жёсткостью: она мягче возвращается к целевой позиции и дольше «колышется», чем жёсткая (`StiffnessHigh`). В модуле она применяется в `expandVertically().usingSpring(Spring.StiffnessLow)` для раскрытия контента.

**Почему не другие:**
- B — 300 мс это `Normal`, а низкая жёсткость не означает «без анимации»
- C — 500 мс это `Slow`; жёсткость — ключевой параметр пружины
- D — резкость даёт как раз высокая жёсткость, а не низкая

</details>

---

## Вопрос 7. Что верно про `ScreenState` и `ScreenEvent` в MVI-части модуля?

- **A)** `ScreenState` — неизменяемый `data class` (`items`, `isLoading`, `error`, `searchQuery`), а `ScreenEvent` — `sealed class` пользовательских действий (`Refresh`, `Search`, `SelectItem`)
- **B)** `ScreenState` — изменяемый класс с `mutableListOf`, а `ScreenEvent` — `enum class`
- **C)** `ScreenState` хранит только ошибки, а события приходят через `StateFlow`
- **D)** Всё наоборот: `sealed class` описывает состояние, а `data class` — события

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Классическая UDF/MVI-структура из content.md: состояние — один неизменяемый `data class`, обновляется только через `copy(...)` в `_state.update { }`. События — `sealed class`, что даёт исчерпывающий `when` в `handleEvent`. Разделение важно: состояние можно считывать многократно (StateFlow), а события — это «факты действия», которые обрабатываются по одному разу.

**Почему не другие:**
- B — изменяемость state разрушает главный принцип UDF: состояние меняется только в ViewModel; `enum` не подходит для событий с данными (`Search(val query: String)`)
- C — `error` — лишь одно из полей; события идут через отдельный `SharedFlow`, а не через `state`
- D — перепутано: `data class` = состояние, `sealed class` = события

</details>

---

## Вопрос 8. Как UI отправляет события в `ScreenViewModel`?

```kotlin
private val _events = MutableSharedFlow<ScreenEvent>()
val events: SharedFlow<ScreenEvent> = _events

init {
    viewModelScope.launch {
        _events.collect { event -> handleEvent(event) }
    }
    loadItems()
}
```

- **A)** В Composable: `onEvent = { event -> viewModel._events.tryEmit(event) }` — в тот же `SharedFlow`, который собирает ViewModel
- **B)** Через `_state.update { }` — состояние и есть канал событий
- **C)** Composable напрямую вызывает `repository.getItems()`
- **D)** Через `startActivity()` с intent-экстрасами

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В коде `Screen` получает `val state by viewModel.state.collectAsState()` для чтения состояния и передаёт в `ScreenContent` колбэк `onEvent`, который делает `viewModel._events.tryEmit(event)`. ViewModel в `init` запускает корутину, собирающую этот поток и вызывающую `handleEvent`. `tryEmit` — неблокирующая отправка без `suspend`.

**Почему не другие:**
- B — `state` предназначен только для чтения UI; это «что показать», а не «что сделать»
- C — нарушение слоёв: Composable не должен знать о repository, вся логика в ViewModel
- D — Android-specific mechanism, здесь не используется и не работает в commonMain

</details>

---

## Вопрос 9. ⚠️ Вопрос-ловушка. Посмотри на обработку выбора элемента:

```kotlin
private suspend fun handleEvent(event: ScreenEvent) {
    when (event) {
        is ScreenEvent.Refresh -> loadItems()
        is ScreenEvent.Search -> searchItems(event.query)
        is ScreenEvent.SelectItem -> selectItem(event.itemId)
    }
}

private suspend fun selectItem(itemId: String) {
    // Emit event для parent screen
    _events.emit(ScreenEvent.SelectItem(itemId))
}
```

Что произойдёт, если UI вызовет `onEvent(ScreenEvent.SelectItem("42"))`?

- **A)** Событие один раз уйдёт родительскому экрану — всё корректно
- **B)** Возникнет бесконечный цикл: ViewModel снова collect то же событие → `handleEvent` → `selectItem` → `emit` → снова ...
- **C)** Событие тихо потеряется: у `SharedFlow` нет подписчиков вне ViewModel
- **D)** Безусловно упадёт приложение с `ClassCastException`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Это реальная ловушка в приведённом коде. `_events` собирается **той же** ViewModel в `init`. Когда приходит `SelectItem`, `handleEvent` вызывает `selectItem`, который эмитит `ScreenEvent.SelectItem` обратно в `_events`. Это событие снова собирается → снова `handleEvent` → снова `selectItem` → ... Получается самоподдерживающийся цикл.

Комментарий в коде говорит «emit event для parent screen», но для этого нужен **отдельный** канал эффектов/навигации (например, другой `SharedFlow<UiEffect>` или колбэк наружу), а не тот же поток входящих событий. У `SharedFlow` с одним и тем же collect внутри того же обработчика зацикливание закономерно.

**Почему не другие:**
- A — один раз не получится: эмитёр и подписчик здесь замкнуты на одном и том же потоке
- C — как раз подписчик есть (корутина в `init`), поэтому событие не теряется, а возвращается
- D — типы совпадают (`ScreenEvent`), `ClassCastException` здесь не откуда взяться; проблема логическая, а не в приведении

</details>

---

## Вопрос 10. В `ScreenContent` есть такой `when`:

```kotlin
when {
    state.isLoading && state.items.isEmpty() -> { /* CircularProgressIndicator */ }
    state.error != null -> { /* Text + Button "Retry" */ }
    state.items.isEmpty() -> { /* Text "No items found" */ }
    else -> { /* LazyColumn со списком */ }
}
```

При каком состоянии пользователь увидит кнопку **Retry**?

- **A)** При `state.error != null` (ветка идёт до проверки `items.isEmpty()`)
- **B)** При `state.isLoading && state.items.isEmpty()`
- **C)** При `state.items.isEmpty()` и отсутствии ошибки
- **D)** Всегда, когда в списке есть элементы

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`when` проверяет ветки по порядку и выполняет первую подходящую. Ошибка проверяется **второй**, сразу после «загрузка + пусто». Значит при `error != null` (и не выполняется первая ветка) покажется `Text("Error: ...")` и `Button(onClick = { onEvent(ScreenEvent.Refresh) })` с надписью Retry. Важно, что ветка error идёт **до** `items.isEmpty()`, иначе при ошибке пустой список перекрыл бы сообщение об ошибке.

**Почему не другие:**
- B — это первая ветка: там показывается `CircularProgressIndicator`, а не Retry
- C — это третья ветка: «No items found» без кнопки
- D — если items есть, выполняется `else` со списком; Retry там нет

</details>

---

## Вопрос 11. Что делает расширение из модуля?

```kotlin
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

- **A)** Бесконечно вращает модификатор от 0° до 360° за 1000 мс, каждый раз начиная сначала
- **B)** Поворачивает один раз на 360° и останавливается
- **C)** Вращает туда-обратно (0 → 360 → 0), потому что `RepeatMode.Restart`
- **D)** Анимирует прозрачность, а не поворот

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`rememberInfiniteTransition()` создаёт бесконечно работающий transition. `animateFloat(0f → 360f)` с `infiniteRepeatable(tween(1000), RepeatMode.Restart)` означает: один оборот занимает 1000 мс, после достижения 360° значение **сбрасывается на 0f** и повторяется. `this.rotate(rotation)` применяет угол к `Modifier`. В `ScreenContent` это используется для иконки обновления, пока `state.isLoading == true`.

**Почему не другие:**
- B — `infiniteRepeatable` специально делает цикл бесконечным
- C — туда-обратно даёт `RepeatMode.Reverse`, а `Restart` перезапускает с начала
- D — за прозрачность отвечал бы `alpha(...)`/`fadeIn`, а здесь явно `rotate(rotation)`

</details>

---

## Вопрос 12. Фрагмент интеграции со SwiftUI:

```kotlin
// commonMain
expect fun createSettingsScreen(
    onSettingChanged: (String, String) -> Unit
): PlatformView

// iosMain
actual fun createSettingsScreen(
    onSettingChanged: (String, String) -> Unit
): PlatformView {
    val settingsViewController = SettingsViewController(onSettingChanged)
    return IOSPlatformView(settingsViewController)
}
```

Как SwiftUI-экран попадает в общий UI-код?

- **A)** Через `expect/actual`: `commonMain` объявляет контракт, `iosMain` возвращает `IOSPlatformView`, внутри которого `UIViewController` с `UIHostingController(rootView: SettingsView(...))`
- **B)** Compose Multiplatform сам рендерит SwiftUI через `ComposeUIViewController`
- **C)** Через GraphQL-запрос, который возвращает готовый экран
- **D)** Через SQLDelight-таблицу с описанием view

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В `commonMain` живёт только объявление `expect fun createSettingsScreen(...): PlatformView` — платформо-независимый контракт с колбэком. В `iosMain` `actual` создаёт `SettingsViewController` и оборачивает его в `IOSPlatformView`. Сам `SettingsViewController` (в коде — subclass `UIViewController`) в `viewDidLoad()` создаёт `UIHostingController(rootView: SettingsView(onSettingChanged: ...))` — так SwiftUI-дерево встраивается в UIKit-контроллер, а тот — в Compose через `PlatformView`.

**Почему не другие:**
- B — `ComposeUIViewController` — это мост в обратную сторону (Compose → iOS); здесь же нативная SwiftUI-вью встраивается в общее приложение через expect/actual
- C и D — сеть и БД не имеют отношения к встраиванию UI; это был бы абсурдный способ передать экран

</details>

---

## Вопрос 13. В `SettingsViewController` SwiftUI-вью подключается так:

```swift
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
```

Зачем здесь `translatesAutoresizingMaskIntoConstraints = false`, четыре `NSLayoutConstraint` и `addChild`/`didMove`?

- **A)** Чтобы SwiftUI-вью заняло весь контейнер через Auto Layout и корректно вошло в иерархию родительского `UIViewController` с соблюдением жизненного цикла
- **B)** Чтобы включить анимации `AnimatedVisibility` внутри SwiftUI
- **C)** Чтобы скрыть клавиатуру при показе экрана
- **D)** Чтобы задать жёсткий размер вью 100×100 pt

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`translatesAutoresizingMaskIntoConstraints = false` отключает автоматическую генерацию констрейнтов из autoresizing mask — иначе ручные констрейнты конфликтовали бы. Четыре якоря (top/leading/trailing/bottom) прижимают вью hosting-контроллера ко всем сторонам родителя. `addChild` + `didMove(toParent:)` — стандартный UIKit-протокол «дочерний контроллер», чтобы родитель корректно управлял жизненным циклом и событиями.

**Почему не другие:**
- B — анимации Compose тут ни при чём; этот код про встраивание в UIKit
- C — скрытие клавиатуры делается через `endEditing(true)` и т.п., здесь этого нет
- D — констрейнты прижимают к краям, а не задают фиксированный размер

</details>

---

## Вопрос 14. Как изменение темы в SwiftUI `SettingsView` передаётся обратно в Kotlin?

```swift
Picker("Theme", selection: $theme) {
    Text("Light").tag("light")
    Text("Dark").tag("dark")
}
.onChange(of: theme) { newValue in
    onSettingChanged("theme", newValue)
}
```

- **A)** Через колбэк `onSettingChanged("theme", newValue)`, объявленный в `expect`-функции как `(String, String) -> Unit`
- **B)** Через `StateFlow` во ViewModel, который SwiftUI подписывает напрямую
- **C)** Через GraphQL `subscription`
- **D)** Через `UserDefaults`, откуда Kotlin читает значение при старте

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`@State private var theme = "light"` хранит выбор внутри SwiftUI. `Picker` с `$theme` обновляет состояние, а `.onChange(of: theme)` перехватывает каждое изменение и вызывает `onSettingChanged("theme", newValue)`. Этот колбэк — тот самый параметр `(String, String) -> Unit`, который пробрасывается из `commonMain` через `expect/actual` в `SettingsViewController` и дальше в `SettingsView`. Так нативная вью уведомляет общий код без любой платформенной зависимости в common-слое.

**Почему не другие:**
- B — SwiftUI не подписывается на Kotlin `StateFlow` напрямую; в коде используется именно замыкание-колбэк
- C — GraphQL-подписки относятся к сетевым данным, а не к локальному UI-событию
- D — `UserDefaults` в этом фрагменте не упоминается; состояние живёт в `@State` и передаётся колбэком

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 13–14 | 🎉 Отлично! Паттерны custom layout, анимаций, MVI и SwiftUI-моста у тебя в голове |
| 10–12 | 📖 Хорошо — перечитай вопросы, где ошибся, особенно про `MeasurePolicy` и событийный поток |
| 6–9 | ⚠️ Пройди модуль ещё раз: сфокусируйся на MVI (`state` vs `events`) и на `expect/actual` для UI |
| < 6 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🔜 Следующий модуль — продвинутая работа с данными: GraphQL, multi-layer кэш и real-time синхронизация. Там будет ещё больше кода и ловушек.

# 📝 Самопроверка: Модуль 3 — Продвинутый Compose Multiplatform

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 10+ из 14 без подглядывания.

---

## Вопрос 1. Зачем, согласно введению модуля, нужен кастомный `Layout`?

- **A)** Для Masonry Grid и сеток с элементами разной высоты, когда `Column` / `Row` / `Box` не дают нужной гибкости
- **B)** Чтобы полностью убрать recomposition из приложения
- **C)** Чтобы заменить `LazyColumn` в любом списке
- **D)** Для настройки анимаций перехода между экранами

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Во введении прямо названы кейсы: Masonry Grid (как в Pinterest), сложная сетка с элементами разной высоты и ситуации, когда стандартные компоненты не дают нужной гибкости. Решение — `Layout` с полным контролем над measure и place.

**Почему не другие:**
- B — recomposition нельзя «убрать», можно лишь сделать её дешевле
- C — `LazyColumn` остаётся лучшим выбором для больших списков; `Layout` — про другую задачу
- D — анимации — отдельная тема раздела 6

</details>

---

## Вопрос 2. Разбери код кастомного Layout из модуля. В каком порядке выполняются действия?

```kotlin
Layout(modifier = modifier, content = content) { measurables, constraints ->
    val placeables = measurables.map { it.measure(constraints) }
    val totalWidth = placeables.sumOf { it.width }
    val totalHeight = placeables.maxOfOrNull { it.height } ?: 0
    layout(totalWidth, totalHeight) {
        placeables.forEach { it.place(x, y) }
    }
}
```

- **A)** Сначала `measure` детей → затем `layout(width, height)` → внутри него `place`
- **B)** Сначала `place` → затем `measure` → затем `layout`
- **C)** `MeasurePolicy` — это сам `Modifier`, порядок не важен
- **D)** Сначала вычисляется размер контейнера, и только потом измеряются дети

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Лямбда `{ measurables, constraints -> ... }` и есть **MeasurePolicy** — политика измерения и размещения. Порядок из модуля:

1. Измеряем детей: `measurable.measure(constraints)` → `Placeable`
2. Вычисляем размеры самого Layout'а
3. Вызываем `layout(width, height) { ... }` и внутри размещаем детей через `place(x, y)`

**Почему не другие:**
- B — нельзя разместить то, что ещё не измерено (нет `width`/`height`)
- C — `Modifier` передаётся отдельным параметром, это не MeasurePolicy
- D — размер контейнера считается **из** размеров уже измеренных детей

</details>

---

## Вопрос 3. Какие поля содержит `Constraints` в примере из модуля?

- **A)** `minWidth`, `maxWidth`, `minHeight`, `maxHeight`
- **B)** `width`, `height`, `x`, `y`
- **C)** `columns`, `spacing`
- **D)** `start`, `end`, `top`, `bottom`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле дано определение:

```kotlin
data class Constraints(
    val minWidth: Int,
    val maxWidth: Int,
    val minHeight: Int,
    val maxHeight: Int
)
```

Именно `constraints.maxWidth` затем используется в MasonryLayout для расчёта ширины колонки.

**Почему не другие:**
- B — это характеристики `Placeable` (`width`, `height`) и координаты `place(x, y)`
- C — параметры самого `MasonryLayout`
- D — таких полей у `Constraints` нет

</details>

---

## Вопрос 4. Фрагмент распределения по колонкам в MasonryLayout:

```kotlin
placeables.forEachIndexed { index, placeable ->
    val columnIndex = columnsHeights.indexOfMin()
    columnItems[columnIndex].add(placeable)
    columnsHeights[columnIndex] += placeable.height + spacing
}
```

Что делает `indexOfMin()`?

- **A)** Возвращает индекс самой «короткой» колонки — туда и кладётся следующий элемент
- **B)** Возвращает минимальную высоту как число
- **C)** Возвращает индекс самой высокой колонки
- **D)** Сортирует `placeables` по высоте

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`indexOfMin()` — кастомный extension на `List<Int>`, написанный ручным циклом. Он ищет индекс минимального значения в `columnsHeights`, то есть колонку с наименьшей текущей высотой. Сюда и добавляется очередная карточка — так сетка получается «рваной», как в Pinterest. После добавления высота колонки увеличивается на `placeable.height + spacing`.

**Почему не другие:**
- B — возвращается именно **индекс**, а не высота
- C — наоборот, выбралась бы самая заполненная колонка
- D — `placeables` не сортируются, меняется только распределение

</details>

---

## Вопрос 5. Почему в MasonryLayout высота контейнера считается как `columnsHeights.maxOrNull() ?: 0`?

- **A)** Высота контейнера должна равняться самой высокой колонке
- **B)** Чтобы все колонки автоматически выровнялись по высоте
- **C)** Чтобы получить суммарную высоту всех элементов
- **D)** Чтобы получить среднюю высоту колонки

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Каждая колонка растёт независимо, поэтому нижняя граница контейнера определяется самой длинной из них — иначе контент вылезет за пределы Layout'а. Ширина при этом берётся целиком: `totalWidth = constraints.maxWidth`.

**Почему не другие:**
- B — выравнивания по высоте здесь нет, колонки остаются разной длины
- C — сумма дала бы слишком большой контейнер
- D — среднее значение обрезало бы самую длинную колонку

</details>

---

## Вопрос 6. Вопрос-ловушка. Что верно про этот фрагмент MasonryLayout?

```kotlin
// Проверяем, что есть место для колонок
if (constraints.maxWidth < columns * 100) {
    // Если места мало, используем одну колонку
}

val placeables = measurables.map { measurable ->
    val columnConstraints = constraints.copy(
        maxWidth = (constraints.maxWidth / columns) - spacing
    )
    measurable.measure(columnConstraints)
}
```

- **A)** Адаптивность реализована: на узком экране автоматически включается одна колонка
- **B)** Условие есть, но его блок пустой — `columns` не меняется, одна колонка фактически не включается
- **C)** Компилятор выбросит это условие как мёртвый код
- **D)** Вместо `columns` следует проверять `spacing`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Это ловушка на «видимую» адаптивность. Комментарий обещает одну колонку, но внутри `if` — только ещё один комментарий: переменная `columns` нигде не переприсваивается. Измерение ниже всё равно делит ширину на исходное `columns`, поэтому при узком экране карточки станут просто очень узкими. Кстати, в домашнем задании как раз требуется «1 колонка на mobile, 2–3 на desktop» — этот код ещё предстоит дописать.

**Почему не другие:**
- A — это именно то, во что легко поверить, глядя на комментарий
- C — компилятор такое не удаляет: условие ссылается на реальные значения
- D — `spacing` тут ни при чём, проблема в пустом теле ветки

</details>

---

## Вопрос 7. Почему антипаттерном считается такой код?

```kotlin
@Composable
fun ExpensiveScreen(data: List<Item>) {
    val sortedData = data.sortedBy { it.name }   // ❌
    LazyColumn { items(sortedData) { /* ... */ } }
}
```

- **A)** `sortedBy` выполняется при каждой recomposition; решение — `val sortedData = remember(data) { data.sortedBy { it.name } }`
- **B)** `sortedBy` не поддерживается в Kotlin Multiplatform
- **C)** Нужно заменить `remember` на `derivedStateOf`
- **D)** `LazyColumn` не принимает уже отсортированные списки

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле приведена наглядная пара:

```kotlin
// ПЛОХО: Вычисляется при каждой перерисовке
val sortedData = data.sortedBy { it.name }

// ХОРОШО: Вычисляется только когда data меняется
val sortedData = remember(data) { data.sortedBy { it.name } }
```

Ключ `data` означает: пересчитывать сортировку только при изменении самого списка.

**Почему не другие:**
- B — сортировка — чистая функция stdlib, на KMP работает
- C — `derivedStateOf` нужен для производных от **состояния** значений, здесь достаточно `remember(data)`
- D — `items(sortedData)` — нормальный вызов

</details>

---

## Вопрос 8. Что делает `derivedStateOf` в примере `SearchScreen`?

```kotlin
val filteredItems = derivedStateOf {
    if (query.isEmpty()) items
    else items.filter { it.name.contains(query, ignoreCase = true) }
}.value
```

- **A)** Позволяет пересчитывать фильтр только когда меняется результат, а `.value` читает текущее значение
- **B)** Переносит фильтрацию в фоновый поток
- **C)** Кэширует список без всяких ключей и условий
- **D)** Полностью отменяет recomposition экрана

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Комментарий в модуле: «Фильтр вычисляется только когда результат меняется». `derivedStateOf` создаёт производное состояние, Compose сравнивает его результат и не дёргает читателей зря, а `.value` — чтение этого состояния.

**Почему не другие:**
- B — за потоки отвечают coroutines/`flowOn`, не `derivedStateOf`
- C — это не «просто кэш», а именно-state-зависимое вычисление
- D — recomposition остаётся, но становится реже

</details>

---

## Вопрос 9. Зачем в оптимизированном списке указан `key = { it.id }`?

```kotlin
items(items, key = { it.id }, itemContent = { item -> /* ... */ })
```

- **A)** Стабильный ключ позволяет Compose переиспользовать элементы и корректно отслеживать их при скролле и изменениях списка
- **B)** Чтобы автоматически отсортировать элементы по id
- **C)** Чтобы задать фиксированную высоту карточки
- **D)** Чтобы включить анимацию появления

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле прямо помечено: `key = { it.id }, // ✅ Всегда используйте ключи!` Для списка на 10 000 элементов это критично: без ключей Compose не может понять, какой элемент остался на месте, и пересоздаёт больше элементов, чем нужно.

**Почему не другие:**
- B — ключ не сортирует, он идентифицирует
- C — высота задаётся модификаторами/контентой
- D — анимация — задача `AnimatedVisibility`, а не `key`

</details>

---

## Вопрос 10. Каким флагом включается Layout Inspector для профилирования, согласно модулю?

- **A)** `-Pcompose.experimental.enableLayoutInspector=true`
- **B)** `-Dcompose.layout.inspector=on`
- **C)** `-Pcompose.debug.recomposition=true`
- **D)** `--enable-layout-inspector`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаг 1 раздела «Профилирование с Layout Inspector»: запустить приложение с флагом `-Pcompose.experimental.enableLayoutInspector=true`, затем открыть Layout Inspector в Android Studio, найти компоненты с частыми перерисовками и оптимизировать их через `remember` / `derivedStateOf`.

**Почему не другие:**
- B, C, D — таких флагов в модуле нет; это выдуманные варианты

</details>

---

## Вопрос 11. Какой API лучше всего подходит для плавного перехода между режимами `ListScreen` и `GridScreen`?

- **A)** `Crossfade(targetState = mode, animationSpec = tween(300))` вместе с `updateTransition(mode, label = "modeTransition")`
- **B)** `AnimatedVisibility(visible = mode == ScreenMode.List)`
- **C)** `slideOutOfContainer` без `targetState`
- **D)** Обычный `when (mode)` без анимации

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле пример `TransitionScreen` использует `updateTransition(mode, label = "modeTransition")` и `Crossfade` с `tween(300)`, внутри которого `when (currentMode)` возвращает `ListScreen()` или `GridScreen()`. Это и есть плавный crossfade между режимами.

**Почему не другие:**
- B — `AnimatedVisibility` про появление/исчезновение одного элемента
- C — `slideOutOfContainer` — часть `AnimatedContent`, а не переход между режимами
- D — без анимации требование «плавности» не выполняется

</details>

---

## Вопрос 12. Что означает `togetherWith` в этом `transitionSpec`?

```kotlin
slideIntoContainer(
    towards = AnimatedContent.Towards.Start,
    animationSpec = tween(300)
) togetherWith
slideOutOfContainer(
    towards = AnimatedContent.Towards.End,
    animationSpec = tween(300)
)
```

- **A)** Анимации входа и выхода выполняются одновременно
- **B)** Сначала полностью входит новый контент, потом выходит старый
- **C)** Анимация отменяется при конфликте направлений
- **D)** `tween(300)` применяется только к входу

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`togetherWith` — инфиксный combinator из Compose Animations: он объединяет две `EnterTransition`/`ExitTransition` в `TransitionSpec`, чтобы они шли параллельно. Здесь новый экран входит в сторону `Start`, а старый выходит в сторону `End` — получается эффект «продольного слайда».

**Почему не другие:**
- B — для последовательности использовался бы другой combinator
- C — направлений конфликта нет: Start и End заданы явно
- D — `tween(300)` указан в обеих анимациях

</details>

---

## Вопрос 13. Что является точкой входа Desktop-приложения в примере из модуля?

- **A)** `fun main() = application { window(title = "SkillSync Desktop", width = 1200.dp, height = 800.dp) { SkillSyncApp() } }`
- **B)** `@Composable fun SkillSyncApp()` — это просто корневой UI
- **C)** `jsBrowserProductionWebpack` — это Web-сборка
- **D)** `actual fun getPlatformName(): String = "Desktop"`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В `apps/skillsync-desktop/src/jvmMain/kotlin/Main.kt` точка входа — обычная `fun main()`, внутри которой вызывается `application { window(...) { SkillSyncApp() } }`. В Gradle это подкреплено настройкой `mainClass = "MainKt"`. `SkillSyncApp()` уже внутри рисует `MaterialTheme` → `Surface` → `AppNavHost()` из commonMain.

**Почему не другие:**
- B — это `@Composable`-функция, без `main` приложение не запустится
- C — относится к Web-версии
- D — платформенный `actual`, а не точка входа

</details>

---

## Вопрос 14. Какая команда собирает production-версию Web-приложения и куда попадают файлы?

- **A)** `./gradlew :apps:skillsync-web:jsBrowserProductionWebpack` → `apps/skillsync-web/build/dist/compileSync/js/mainProduction`
- **B)** `./gradlew assembleRelease` → `build/outputs/apk`
- **C)** `./gradlew :apps:skillsync-desktop:package` → `build/install`
- **D)** `./gradlew jsNodeProductionWebpack` → `build/dist/js/nodeProduction`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Раздел «Публикация на GitHub Pages» говорит: собрать Web-версию командой `./gradlew :apps:skillsync-web:jsBrowserProductionWebpack`, файлы окажутся в `apps/skillsync-web/build/dist/compileSync/js/mainProduction`, после чего их загружают в репозиторий и включают GitHub Pages. Таргет при этом настраивается как `js(IR) { browser(); binaries.executable(); ... moduleKind = "umd" }`.

**Почему не другие:**
- B — это Android-сборка
- C — это Desktop-упаковка
- D — node-таргет здесь не используется, в модуле — `browser()`

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 12–14 | 🎉 Отлично! Кастомные Layout'ы, перформанс и анимации — ты готов к сложным UI-задачам |
| 9–11 | 📖 Перечитай секции про оптимизацию списка и про Desktop/Web-таргеты |
| 5–8 | ⚠️ Пройди модуль ещё раз: сфокусируйся на MeasurePolicy, MasonryLayout и `remember`/`derivedStateOf` |
| < 5 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🔜 Следующий модуль Senior Track — продвинутая работа с данными: GraphQL, real-time синхронизация и сложные стратегии кэширования.

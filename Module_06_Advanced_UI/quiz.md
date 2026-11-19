# 📝 Самопроверка: Модуль 6

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 10+ из 13 без подглядывания.

---

## Вопрос 1. Какая библиотека рекомендуется в модуле для графиков в KMP и почему?

- **A)** MPAndroidChart — у неё больше всего типов графиков
- **B)** Vico — Compose-first, полная KMP-поддержка, оптимизация под 60 FPS и гибкая кастомизация
- **C)** Canvas API — потому что всё придётся рисовать вручную
- **D)** Стандартные Charts из `material3` — встроены в Compose

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В таблице модуля Vico помечен ✅ «Рекомендуется»: нативный Compose, KMP-поддержка, производительность (60 FPS), полный контроль над стилями. MPAndroidChart — для сложных дашбордов, но с трудом интегрируется с Compose; Canvas даёт полный контроль, но требует рисовать всё вручную.

**Почему другие не подходят:**
- A — в модуле отмечена как вариант «для сложных дашбордов», не как основная рекомендация
- C — это fallback для кастомных визуализаций
- D — такой встроенной библиотеки в модуле нет

</details>

---

## Вопрос 2. Как `ActivityChart` передаёт новые данные в Vico?

```kotlin
val modelProducer = remember { CartesianChartModelProducer() }
LaunchedEffect(dataPoints) {
    if (dataPoints.isNotEmpty()) {
        modelProducer.runTransaction {
            lineSeries { series(dataPoints) }
        }
    }
}
```

- **A)** Через `StateFlow`, который Vico подписывает сам
- **B)** Создаётся `CartesianChartModelProducer` один раз через `remember`, а обновление происходит в `LaunchedEffect(dataPoints)` транзакцией `runTransaction { lineSeries { series(...) } }`
- **C)** Путём пересоздания `CartesianChartHost` при каждом изменении списка
- **D)** Через `mutableStateOf` внутри `modelProducer`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`CartesianChartModelProducer` — это «движок данных» графика. Он создаётся один раз (`remember`), а при изменении `dataPoints` `LaunchedEffect` запускает транзакцию, в которой `lineSeries { series(dataPoints) }` задаёт точки lineseries. `CartesianChartHost` получает `modelProducer = modelProducer` и перерисовывается сам.

**Почему другие не подходят:**
- A — Vico не подписывается на Flow автоматически
- C — Host не пересоздаётся, меняется модель
- D — обновление идёт через транзакцию, а не через состояние внутри producer

</details>

---

## Вопрос 3. Что произойдёт при вызове `ActivityChart(dataPoints = emptyList())`?

- **A)** Отобразится график с нулевыми значениями
- **B)** Функция сразу вернётся (`if (dataPoints.isEmpty()) return`) — график не будет отрисован
- **C)** Приложение упадёт с `IndexOutOfBoundsException`
- **D)** Будет показана заглушка «Нет данных»

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В самой первой строке компонента стоит ранний выход:

```kotlin
if (dataPoints.isEmpty()) return
```

Никакой заглушки в коде нет — пустой график просто не рисуется. Если нужна заглушка, её придётся добавить отдельно.

**Почему другие не подходят:**
- A — `lineSeries { series(emptyList()) }` не вызывается из-за раннего возврата
- C — исключения не будет, данные даже не попадут в producer
- D — никакой заглушки в модуле не описано

</details>

---

## Вопрос 4. Какие оси используются в `ActivityChart`?

- **A)** `startAxis = HorizontalAxis.rememberBottom()`, `bottomAxis = VerticalAxis.rememberStart()`
- **B)** `startAxis = VerticalAxis.rememberStart()` (левая вертикальная), `bottomAxis = HorizontalAxis.rememberBottom()` (нижняя горизонтальная)
- **C)** Только `bottomAxis`, вертикальная ось отключена
- **D)** Оси создаются автоматически, код их не настраивает

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В `rememberCartesianChart(...)` явно передаются обе оси:

```kotlin
startAxis = VerticalAxis.rememberStart(
    line = rememberLineComponent(fill = Fill(Color.Gray), thickness = 2.dp),
    guideline = rememberLineComponent(fill = Fill(Color.LightGray.copy(alpha = 0.3f)), thickness = 1.dp)
),
bottomAxis = HorizontalAxis.rememberBottom(
    line = rememberLineComponent(fill = Fill(Color.Gray), thickness = 2.dp),
    guideline = null
)
```

`start` = левая сторона, `bottom` = нижняя. У нижней оси `guideline = null`, у левой — светлая сетка.

**Почему другие не подходят:**
- A — типы осей перепутаны местами
- C — `startAxis` задан явно
- D — настройка осей — существенная часть кода компонента

</details>

---

## Вопрос 5. Каков размер графика по умолчанию, который задаёт `ActivityChart` в `CartesianChartHost`?

- **A)** `fillMaxWidth()` и `height(120.dp)`
- **B)** `fillMaxWidth()` и `height(200.dp)`
- **C)** `fillMaxSize()` и `height(300.dp)`
- **D)** Размер не задан, график занимает только содержимое

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В конце `CartesianChartHost`:

```kotlin
modifier = modifier
    .fillMaxWidth()
    .height(200.dp)
```

Высота фиксированная — 200.dp, ширина — на весь доступный контейнер. В `StatisticsScreen` к этому добавляется `padding(vertical = 16.dp)`.

**Почему другие не подходят:**
- A — 120.dp в коде нет
- C — `fillMaxSize()` не используется, высота не 300.dp
- D — модификатор явно задаёт размеры

</details>

---

## Вопрос 6. Что бросается в глаза в объявлении:

```kotlin
@Composable
@Composable
fun StatisticsScreen(
    weeklyData: List<Float> = listOf(5f, 8f, 12f, 7f, 15f, 10f, 18f),
    monthlyData: List<Float> = listOf(35f, 42f, 38f, 45f),
    onBack: () -> Unit
) { ... }
```

- **A)** Всё правильно: две аннотации усиливают эффект
- **B)** `@Composable` указан дважды — достаточно одной аннотации, дубль избыточен (аннотация не является повторяемой)
- **C)** Не хватает `@Preview`
- **D)** Параметры должны быть `val`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B** (вопрос-ловушка!)

В файле `ActivityChart.kt` над `StatisticsScreen` аннотация `@Composable` действительно написана дважды. Одна аннотация полностью определяет composable-функцию; дублирование — опечатка, которая не даёт никаких преимуществ и может вызвать ошибку компиляции (аннотация не является повторяемой).

**Почему другие не подходят:**
- A — «усиления эффекта» не существует
- C — `@Preview` нужен только для превью в студии, это не ошибка объявления
- D — параметры функции не объявляются с `val`/`var`

</details>

---

## Вопрос 7. Какие три типа анимаций перечислены в теории модуля?

- **A)** `AnimatedVisibility`, `updateTransition`, `animate*AsState`
- **B)** `animateContentSize`, `scale`, `clickable`
- **C)** `fadeIn`, `slideOut`, `tween`
- **D)** `LaunchedEffect`, `remember`, `StateFlow`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Раздел 3 («Анимации в Compose») перечисляет:
1. **AnimatedVisibility** — показ/скрытие с анимацией
2. **updateTransition** — плавное изменение параметров
3. **animate\*AsState** — анимация значений

`tween(300)` — это способ задать длительность/кривую, а не отдельный тип анимации; `fadeIn`/`slideIntoContainer` применяются для переходов в навигации.

**Почему другие не подходят:**
- B — это модификаторы кода карточки, не список типов из теории
- C — конкретные эффекты и настройка длительности
- D — это состояния и эффекты Compose, не анимации

</details>

---

## Вопрос 8. Что делает эта часть `AchievementCard`?

```kotlin
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
    ...
)
```

- **A)** При нажатии карточка увеличивается до 1.05f
- **B)** При нажатии карточка сжимается до 0.95f, а `animateContentSize(tween(300))` плавно анимирует изменение её размера
- **C)** Карточка всегда имеет масштаб 0.95f
- **D)** `isPressed` здесь не влияет на внешний вид

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`scale = if (isPressed) 0.95f else 1f` — эффект «нажатия» (scale down). `animateContentSize(tween(300))` анимирует изменение размеров контента за 300 мс. Значение `isPressed` меняется в `clickable` и напрямую влияет на модификатор `scale`.

**Почему другие не подходят:**
- A — значение 0.95f меньше единицы, это сжатие
- C — вне нажатия масштаб равен 1f
- D — состояние напрямую подставлено в `scale`

</details>

---

## Вопрос 9. В каком случае в `AchievementCard` отображается `LinearProgressIndicator`?

- **A)** Всегда — прогресс-бар есть у каждого бейджа
- **B)** Только если бейдж разблокирован
- **C)** Только если `!achievement.isUnlocked && achievement.progress > 0`
- **D)** Только при нажатии на карточку

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

Условие в коде:

```kotlin
if (!achievement.isUnlocked && achievement.progress > 0) {
    LinearProgressIndicator(
        progress = achievement.progress,
        ...
    )
}
```

Смысл: разблокированным бейджам прогресс не нужен, а при нулевом прогрессе полоса не показывает ничего полезного.

**Почему другие не подходят:**
- A — Condition явно ограничивает отображение
- B — наоборот: для разблокированных он скрыт
- D — состояние нажатия здесь не участвует

</details>

---

## Вопрос 10. Какие направления слайдов заданы в `AnimatedNavHost`?

- **A)** `enterTransition` — вправо, `exitTransition` — влево, `popEnter` — влево, `popExit` — вправо
- **B)** `enter` / `exit` — `SlideDirection.Left`, `popEnter` / `popExit` — `SlideDirection.Right`
- **C)** Все переходы используют `SlideDirection.Left`
- **D)** Направления не заданы, используется только `fadeIn`/`fadeOut`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

При переходе вперёд (`enter`/`exit`) экран «уезжает» влево: `slideIntoContainer(SlideDirection.Left) + fadeIn`, `slideOutOfContainer(SlideDirection.Left) + fadeOut`. При возврате (`popEnter`/`popExit`) направление меняется на `Right` — так пользователь видит обратное движение. Все анимации — `tween(300)`.

**Почему другие не подходят:**
- A — направления перепутаны
- C — pop-переходы используют Right
- D — слайды заданы явно, fade добавляется к ним

</details>

---

## Вопрос 11. Где лежат файлы дизайн-системы и как называется итоговая тема приложения?

- **A)** В `androidMain/ui/theme`; тема — `MaterialTheme`
- **B)** В `commonMain/kotlin/com/ecotrack/ui/theme/` — `Colors.kt`, `Typography.kt`, `Theme.kt`; тема — `EcoTrackTheme`
- **C)** В `iosMain/ui/theme`; тема — `EcoTrackTypography`
- **D)** В `commonTest` — дизайн-система тестируется, а не используется в UI

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Модуль создаёт три файла в `commonMain`: `Colors.kt` (светлые/тёмные/общие цвета), `Typography.kt` (`EcoTrackTypography` со стилями текста) и `Theme.kt`. composable-обёртка называется `EcoTrackTheme(darkTheme, content)` и внутри вызывает `MaterialTheme(colorScheme = colors, typography = EcoTrackTypography, content = content)`. Общий код доступен всем платформам.

**Почему другие не подходят:**
- A — дизайн-система намеренно лежит в commonMain, а не только на Android
- C — `EcoTrackTypography` это типографика, не тема
- D — тесты тут ни при чём

</details>

---

## Вопрос 12. Найди ошибку во фрагменте `Theme.kt`:

```kotlin
val LightEcoTrackColors = darkColorScheme(
    primary = LightPrimary,
    secondary = LightSecondary,
    background = LightBackground,
    surface = LightSurface,
    error = LightError
)

val DarkEcoTrackColors = lightColorScheme(
    primary = DarkPrimary,
    secondary = DarkSecondary,
    background = DarkBackground,
    surface = DarkSurface,
    error = DarkError
)
```

- **A)** Ошибки нет: светлая схема создаётся через `darkColorScheme`, это требуется Material 3
- **B)** Перепутаны функции: светлая палитра создана через `darkColorScheme`, а тёмная — через `lightColorScheme`
- **C)** Не хватает аннотации `@Composable` у `val`
- **D)** Цвета `LightPrimary` и `DarkPrimary` вообще не определены в `Colors.kt`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B** (вопрос-ловушка!)

По смыслу должно быть наоборот:

```kotlin
val LightEcoTrackColors = lightColorScheme(primary = LightPrimary, ...)
val DarkEcoTrackColors = darkColorScheme(primary = DarkPrimary, ...)
```

`darkColorScheme()` задаёт тёмные базовые роли (например, `onPrimary` по умолчанию тёмный/светлый в соответствии с тёмной темой). Подстановка светлых цветов в «тёмную» схему приведёт к неожиданным контрастам. В `EcoTrackTheme` выбор делается так: `if (darkTheme) DarkEcoTrackColors else LightEcoTrackColors` — то есть ошибка проявится во внешнем виде приложения.

**Почему другие не подходят:**
- A — Material 3 не требует такого «переворота», это явная опечатка в модуле
- C — `val` со схемой не является composable-функцией
- D — все перечисленные цвета определены в `Colors.kt`

</details>

---

## Вопрос 13. Какое правило про длительность анимаций дано в конце модуля и какие инструменты предлагаются для отладки?

- **A)** Анимация длиннее 300ms — слишком медленная; для отладки — «Show layout bounds» и «Debug GPU overlay» в Developer Options
- **B)** Анимация длиннее 1000ms — слишком медленная; для отладки — Charles Proxy
- **C)** Ограничений нет; для отладки — Logcat и `remember`
- **D)** Анимация длиннее 300ms — оптимальная; для отладки — Android Profiler

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Совет модуля: не переусердствуй с анимациями — они должны улучшать UX, а не отвлекать. Правило: **если анимация длится более 300ms — она слишком медленная**. Для отладки предлагается включить в Developer Options Android Studio «Show layout bounds» и «Debug GPU overlay». Обрати внимание: в коде иконки бейджа стоит `animateItemOpacity(tween(500))` — это как раз случай, который стоит пересмотреть по этому правилу.

**Почему другие не подходят:**
- B — порог 1000ms не упоминается, Charles Proxy относится к Модулю 5
- C — правило есть, и инструменты названы конкретно
- D — 300ms — верхняя граница, а не «оптимальная длительность»

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 12–13 | 🎉 Отлично! Графики, анимации и дизайн-система — твоя стихия, Переходи к Модулю 7 |
| 9–11 | 📖 Перечитай разделы 2 и 3: Vico-транзакции и анимации карточки/переходов |
| 6–8 | ⚠️ Пройди модуль ещё раз; сфокусируйся на `CartesianChartModelProducer` и файлах темы |
| < 6 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🔜 Квиз к Модулю 7 (нативные фичи) — про уведомления, deep links и файловую систему.

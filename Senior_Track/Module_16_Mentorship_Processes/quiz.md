# 📝 Самопроверка: Модуль 16 — Mentorship & Professional Growth

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 12+ из 15 без подглядывания.

---

## Вопрос 1. Что из этого **является нарушением** согласно KMP Code Review Checklist?

- **A)** Platform-specific код изолирован в `expect`/`actual`
- **B)** Прямые вызовы Android/iOS SDK в `commonMain`
- **C)** `StateFlow`/`SharedFlow` используются по назначению
- **D)** Обработка ошибок через `Result` type

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В чек-листе прямо есть пункт «No direct Android/iOS SDK calls in commonMain» и «Common code не содержит platform-specific logic». Всё остальное — наоборот, требования чек-листа: изоляция через `expect`/`actual`, правильные Flow и `Result`.

**Почему другие не подходят:**
- A — это как раз правильный паттерн
- C — рекомендуется в разделе Architecture & Design
- D — тоже рекомендуется («Error handling с Result type»)

</details>

---

## Вопрос 2. Ловушка. На ревью middle-разработчик показывает код и говорит: «Всё ок, у нас же Koin для DI». Что ответить?

```kotlin
// commonMain — WRONG!
class UserManager {
    fun getUserInfo(): String {
        val context = androidContext() // ❌
        val prefs = context.getSharedPreferences("user", 0)
        return prefs.getString("name", "Unknown") ?: "Unknown"
    }
}
```

- **A)** Это нарушение: в `commonMain` не должно быть platform-specific вызовов. Нужен контракт `PlatformStorage` и `actual`-реализации; DI сам по себе проблему не решает
- **B)** Всё действительно ок: Koin подставит зависимости, разбирать нечего
- **C)** Достаточно заменить `getSharedPreferences` на `nonNull()` — и код станет кроссплатформенным
- **D)** Проблема только в отсутствии KDoc, надо дописать документацию

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Это вопрос-ловушка на подмену понятий. `androidContext()` — Android API, его физически нет на iOS, и никакой Koin не сделает его доступным в `commonMain`. DI решает, **как получить зависимость**, но не решает, **где живёт платформенный код**. Правильный пример из модуля:

```kotlin
// commonMain
interface PlatformStorage {
    fun getString(key: String, defaultValue: String): String
    fun putString(key: String, value: String)
}

class UserManager(private val storage: PlatformStorage) {
    fun getUserInfo(): String = storage.getString("name", "Unknown")
}
```

А уже в `androidMain` и `iosMain` — `actual class PlatformStorage` с `SharedPreferences` и `NSUserDefaults`.

**Почему другие не подходят:**
- B — типичная ловушка «у нас есть DI, значит можно»; DI не изолирует API от компиляции
- C — `nonNull()`/`requireNotNull()` нужны для проверки nullable, а не для кроссплатформенности
- D — документация тут ни при чём, код не соберётся под iOS

</details>

---

## Вопрос 3. В правильном примере с `PlatformStorage` — что используется в реализациях для Android и iOS?

- **A)** `androidMain`: `SharedPreferences` через `androidContext()`; `iosMain`: `NSUserDefaults.standardUserDefaults`
- **B)** Обе реализации лежат в `commonMain` и различаются только `if (isAndroid)`
- **C)** `androidMain`: `NSUserDefaults`; `iosMain`: `SharedPreferences`
- **D)** Реализации задаются в `gradle.properties` как строки

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В content.md показаны две `actual class PlatformStorage actual constructor()`:
- Android: `private val prefs = androidContext().getSharedPreferences("user", 0)`, запись через `prefs.edit().putString(...).apply()`
- iOS: `private val defaults = NSUserDefaults.standardUserDefaults`, чтение `defaults.stringForKey(key) ?: defaultValue`, запись `defaults.setObject(value, key)`

Общий слой видит только интерфейс `PlatformStorage`.

**Почему другие не подходят:**
- B — `if (isAndroid)` в general-коде как раз и есть антипаттерн, с которым борется чек-лист
- C — перепутаны платформы: `SharedPreferences` — Android, `NSUserDefaults` — iOS
- D — в `gradle.properties` хранят конфигурацию сборки, а не реализацию интерфейса

</details>

---

## Вопрос 4. Ты открываешь PR в шаблоне `.github/PULL_REQUEST_TEMPLATE.md`. Какие опции предлагает блок «Platforms Affected»?

- **A)** Common code, Android-specific, iOS-specific
- **B)** Только Android и iOS
- **C)** Frontend, Backend, Infrastructure
- **D)** Major, Minor, Patch

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаблон содержит:

```markdown
## 📱 Platforms Affected
- [ ] Common code
- [ ] Android-specific
- [ ] iOS-specific
```

Это заставляет автора явно отметить зону влияния — критично для KMP, где изменение в common-коде затрагивает обе платформы, а platform-specific — только одну.

**Почему другие не подходят:**
- B — забыт common-код, самый важный пункт
- C — это не разделы шаблона PR
- D — это типы изменений из блока «Type of Change» (`Bug fix`, `New feature`, `Breaking change`, `Documentation update`), причём даже там формулировка другая

</details>

---

## Вопрос 5. Практика: ты менторишь Junior. Что из этого **НЕ** входит в рекомендованный для него набор?

- **A)** Code review с подробными объяснениями
- **B)** Pair programming 1–2 раза в неделю
- **C)** Technical decision ownership
- **D)** Еженедельные 1:1 встречи и чёткие задачи с acceptance criteria

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C**

По фреймворку из модуля для **Junior**: подробный code review, pair programming 1–2 раза в неделю, понятные задачи с критериями приёмки и регулярные еженедельные 1:1.

`Technical decision ownership` — это пункт для **Middle** разработчиков. Давать его junior слишком рано.

**Почему другие не подходят:**
- A, B, D — всё это прямо перечислено в списке для Junior

</details>

---

## Вопрос 6. Практика: middle-разработчик просит у тебя «больше ответственности». Что из этого соответствует его уровню по фреймворку?

- **A)** Архитектурные дискуссии, ревью с фокусом на design patterns, reverse mentorship junior'ов и технические решения
- **B)** Только подробные объяснения в ревью и еженедельные 1:1
- **C)** Стратегическое планирование и discussion roadmaps
- **D)** Facilitation кросс-командного взаимодействия и управление техдолгом

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Список для **Middle** в content.md: architectural discussions, code review с фокусом на design patterns, mentorship junior developers (reverse mentorship), technical decision ownership. Это логический следующий шаг после junior-пакета.

**Почему другие не подходят:**
- B — это набор для Junior
- C и D — это уже уровень Senior (strategic planning, cross-team collaboration, technical debt management)

</details>

---

## Вопрос 7. Практика: senior-инженер просит менторства. Что из этого **НЕ** относится к его уровню?

- **A)** Стратегическое планирование и roadmap-дискуссии
- **B)** Cross-team collaboration facilitation
- **C)** Управление техническим долгом и career development guidance
- **D)** Pair programming 1–2 раза в неделю с подробным разбором каждой строки

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: D**

Для **Senior** фреймворк предлагает: стратегическое планирование и roadmap, facilitation взаимодействия между командами, управление техдолгом и помощь с карьерным ростом. Pair programming с построчным разбором — инструмент уровня Junior; senior'у нужны стратегические и системные темы.

**Почему другие не подходят:**
- A, B, C — все три прямо из списка для Senior

</details>

---

## Вопрос 8. В шаблоне сессии менторства (`Mentorship Session Template`) в разделе «Feedback» есть два подраздела. Какие?

- **A)** `What went well` и `Areas for improvement`
- **B)** `Pros` и `Cons`
- **C)** `Goals for Today` и `Topics Covered`
- **D)** `Action Items` и `Resources Shared`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаблон сессии:

```markdown
## 📝 Feedback
### What went well:
- 
### Areas for improvement:
- 
```

Это важный момент: обратная связь фиксируется в обе стороны — что удалось, что улучшить. Остальные блоки есть в шаблоне, но это другие разделы.

**Почему другие не подходят:**
- B — таких заголовков в шаблоне нет
- C — это начало сессии (цели и темы)
- D — это action items и ссылки для самостоятельной работы

</details>

---

## Вопрос 9. Практика: на 3–4-й сессии менторства junior спрашивает про Koin, SQLDelight и Ktor. К какому блоку дорожной карты это относится?

- **A)** Session 7–8: Advanced Topics
- **B)** Session 1–2: KMP Fundamentals
- **C)** Session 3–4: Common Code Best Practices
- **D)** Session 9–10: Production Readiness

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Дорожная карта из модуля:
- 1–2 — fundamentals, структура, expect/actual
- 3–4 — platform-agnostic код, coroutines/Flow, serialization, тесты
- 5–6 — Android/iOS интеграция, нативные мосты
- **7–8 — Advanced Topics: Koin, SQLDelight, Ktor, Clean/MVVM**
- 9–10 — Production Readiness: CI/CD, перформанс, безопасность, мониторинг

Ловушка в том, что Ktor/SQLDelight — это «продвинутые темы», а не production readiness, хотя они звучат как инфраструктура.

**Почему другие не подходят:**
- B —.basics, сюда входит expect/actual, а не DI/БД/сеть
- C — тут common-код и Flow, но не конкретные библиотеки
- D — туда относят CI/CD и observability (то, о чём был предыдущий модуль)

</details>

---

## Вопрос 10. В ADR в каком разделе фиксируются принятые компромиссы и отрицательные последствия решения?

- **A)** `Consequences` (Positive / Negative)
- **B)** `Status`
- **C)** `References`
- **D)** `Context`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Структура ADR из `docs/adr/001-kmp-architecture.md`: Status, Context, Decision, Rationale, **Consequences** (Positive и Negative), Alternatives Considered, References. Именно в Consequences написано про `More boilerplate code initially` и `Steeper learning curve for juniors` — то есть честная цена решения.

**Почему другие не подходят:**
- B — Status это `✅ Accepted | 🚧 Proposed | ❌ Rejected | ⚠️ Deprecated`
- C — ссылки на документацию
- D — Context описывает задачу и ограничения, а не итог выбора

</details>

---

## Вопрос 11. Практика: в ADR 001 сравниваются два варианта архитектуры. Какой был принят и почему отвергнут другой?

- **A)** Принят Layer-First (Clean Architecture: `domain`, `data`, `presentation`, `features`); Feature-First отвергнут из-за дублирования и сложности переиспользования utilities
- **B)** Принят Feature-First, а Layer-First отвергнут как слишком простой
- **C)** Приняты оба варианта одновременно
- **D)** Оба отвергнуты, выбрали MVVM без слоёв

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В ADR прямо указано:

> Alternative 1: Feature-First — **Rejected because:** Too much duplication, harder to share common utilities.
> Alternative 2: Layer-First Architecture (Selected) — **Accepted because:** Better code sharing, clearer separation of concerns.

Плюсы выбранного: maintainability, онбординг, test coverage. Минусы: больше boilerplate и крутая кривая обучения для junior'ов.

**Почему другие не подходят:**
- B — прямо противоположно тексту ADR
- C — ADR фиксирует один выбор, иначе это не решение
- D — MVVM упоминается как паттерн, но не как альтернатива архитектуре shared-модуля

</details>

---

## Вопрос 12. Команда предлагает «сначала запрограммировать, потом напишем Problem Definition и выберем 3 варианта». Что не так по Technical Decision Process?

- **A)** Порядок нарушен: сначала Problem Definition и ограничения, затем минимум 3 варианта с плюсами/минусами, критерии, решение с rationale, план внедрения и ретроспектива через 3 месяца
- **B)** Ничего, фреймворк допускает любой порядок
- **C)** Достаточно одного варианта — главнее быстрее реализовать
- **D)** Ретроспектива по фреймворку не предусмотрена

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаги из content.md:
1. Problem Definition (проблема и ограничения времени/бюджета/навыков)
2. Options Analysis — **at least 3** вариантов, pros/cons, effort, долгосрочная стоимость
3. Decision Criteria — performance, DX, сложность тестов, расширяемость
4. Decision & Rationale — выбранный вариант и принятые trade-offs
5. Implementation Plan — поэтапный rollout, rollback, метрики успеха
6. Review & Retrospective — встреча через **3 месяца**, фидбек команды, обновление ADR

**Почему другие не подходят:**
- B — фреймворк задаёт чёткую последовательность
- C — минимум 3 варианта — явное требование
- D — ретроспектива через 3 месяца есть в списке

</details>

---

## Вопрос 13. Ловушка. В примере GitHub-профиля Senior KMP Developer в разделе «KMP Projects / Project 1» указаны метрики.Какая из этих метрик там **НЕ** приведена?

- **A)** Code Sharing — 85% между Android и iOS
- **B)** Reduced development time by 40% с помощью KMP
- **C)** Improved app performance by 30%
- **D)** Reduced release time from 2 days to 2 hours

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: D**

Это вопрос-ловушка на смешивание двух разных блоков. Метрика «CI/CD pipeline reducing release time from 2 days to 2 hours» есть в **резюме** (более ранняя работа `Senior Kotlin Multiplatform Developer`), а не в описании Project 1 в GitHub-профиле.

В Project 1 указано: 10K+ downloads, Clean Architecture + Feature Modules, стек KMP/Compose/SQLDelight/Ktor/Koin, 85% shared code, −40% времени разработки, +30% перформанса, команда из 4 человек.

**Почему другие не подходят:**
- A, B, C — все три действительно есть в Project 1

</details>

---

## Вопрос 14. Практика: что из этого **НЕ** является критерием сдачи домашнего задания Модуля 10?

- **A)** 3+ подробных code review с конструктивной обратной связью в open-source KMP проектах
- **B)** Mentorship plan с чётким learning path для junior
- **C)** ADR с корректной документацией и обоснованием решения
- **D)** CI pipeline проходит быстрее 15 минут

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: D**

Критерии сдачи Модуля 10: 3+ detailed code reviews, mentorship plan с clear learning path, ADR с proper documentation и rationale, обновлённое портфолио с KMP-опытом.

«CI pipeline проходит за < 15 минут» — критерий **Модуля 9** (CI/CD & DevOps), который перемешан здесь специально.

**Почему другие не подходят:**
- A, B, C — прямо из критериев сдачи текущего модуля

</details>

---

## Вопрос 15. Тебе дали system design задание: спроектировать кроссплатформенное фитнес-приложение (GPS, пульс, социальные функции, offline-first с синхронизацией, аналитика). Какие точки обсуждения ожидаются?

- **A)** Выбор архитектуры, data layer на SQLDelight + кэш, real-time sync, интеграция нативных сенсоров, оптимизация производительности для real-time данных
- **B)** Только выбор между Compose и XML-разметкой
- **C)** Только сравнение KMP и Flutter
- **D)** Только настройка GitHub Actions

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В модуле перечислены Expected Discussion Points ровно по пяти пунктам:
1. Architecture выбор (Clean, Feature-first)
2. Data layer strategy (SQLDelight + caching)
3. Real-time sync mechanism
4. Native sensor integration
5. Performance optimization для real-time data

Это и есть демонстрация senior-уровня: ты думаешь не только про UI, а про данные, синхронизацию, нативные API и перформанс.

**Почему другие не подходят:**
- B, C, D — каждый вариант сужает задачу до одной технологии и игнорирует требования (GPS/пульс, офлайн, социальные функции, аналитика)

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 13–15 | 🎉 Отлично! Ты готов объяснять свои решения, как senior — пора делать это в реальных PR и ADR |
| 10–12 | 📖 Хорошо. Перечитай секции про уровни менторства и Technical Decision Process |
| 7–9 | ⚠️ Повтори модуль: сфокусируйся на ролях (Junior / Middle / Senior) и структуре ADR |
| < 7 | 🆘 Вернись к чек-листу code review и шаблонам, пройди ещё раз с заметками |

---

> 🎓 Это финальный квиз Senior Track. Дальше — практика, open source и выступления на комьюнити-мероприятиях.

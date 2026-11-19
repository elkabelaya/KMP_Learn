# 📝 Самопроверка: Модуль 8 — Мультиагентность и оркестрация

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 13+ из 15 без подглядывания.

---

## Вопрос 1. Что такое мультиагентная система (Multi-Agent System)?

- **A)** Одна LLM с очень длинным промптом
- **B)** Система из нескольких автономных AI-агентов, взаимодействующих для решения сложной задачи
- **C)** Набор обычных микросервисов без какого-либо AI
- **D)** Один агент, который рекурсивно вызывает сам себя

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Из темы 8.1: **Multi-Agent System** — это система из нескольких автономных AI-агентов, которые взаимодействуют друг с другом для решения сложных задач. Ключевые концепции:

- **Автономность** — каждый агент принимает решения самостоятельно
- **Взаимодействие** — агенты общаются через сообщения
- **Координация** — согласование действий для общей цели
- **Специализация** — каждый агент отвечает за свою область

**Почему другие не подходят:**
- A — один агент с длинным промптом остаётся одним агентом
- C — без AI это просто распределённая система, не мультиагентная
- D — рекурсия одного агента ≠ несколько автономных агентов с взаимодействием

</details>

---

## Вопрос 2. Какие четыре концепции модуль называет ключевыми для мультиагентных систем?

- **A)** Автономность, Взаимодействие, Координация, Специализация
- **B)** Скорость, Кэширование, Логирование, Мониторинг
- **C)** Chains, Agents, Memory, Tools
- **D)** Sequential, Parallel, Hierarchical, Collaborative

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Это ровно тот список из темы 8.1. Важно не путать уровень идей с уровнем реализации:

| Что | Где в модуле |
|-----|--------------|
| Автономность, Взаимодействие, Координация, Специализация | Ключевые **концепции** (8.1) |
| Sequential, Parallel, Hierarchical, Collaborative | **Паттерны взаимодействия** агентов (8.1) |
| Chains, Agents, Memory, Tools | Компоненты **LangChain** (8.2) |

**Почему другие не подходят:**
- B — это инфраструктурные свойства, а не концепции мультиагентности
- C — компоненты LangChain
- D — паттерны взаимодействия, а не базовые концепции

</details>

---

## Вопрос 3. У тебя независимые части задачи, которые можно выполнять одновременно. Какой паттерн подходит?

```kotlin
val results = coroutineScope {
    val a = async { agentA.execute(taskPart1) }
    val b = async { agentB.execute(taskPart2) }
    val c = async { agentC.execute(taskPart3) }
    listOf(a.await(), b.await(), c.await())
}
```

- **A)** Sequential — линейные workflows с чёткой последовательностью
- **B)** Parallel — независимые задачи, выполняемые одновременно
- **C)** Hierarchical — менеджер распределяет задачи между воркерами
- **D)** Collaborative — агенты обсуждают задачу до консенсуса

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Фрагмент почти дословно повторяет паттерн **Parallel** из темы 8.1: `coroutineScope` + `async { ... }` + `await()`. Модуль прямо указывает, когда его использовать: *«Независимые задачи, которые можно выполнять одновременно»*.

Тот же приём используется и в практическом `ParallelFeatureDevelopment` (тема 8.3): `features.map { async { ... } }.awaitAll().toMap()`.

**Почему другие не подходят:**
- A — Sequential это `resultB = agentB.execute(resultA.output)`: каждый шаг ждёт предыдущий
- C — у Hierarchical есть `ManagerAgent`, `distributeTask` и `collectResults`
- D — у Collaborative есть `GroupChat` и `discuss(task) { ... }`

</details>

---

## Вопрос 4. Что делает код `manager.distributeTask(complexTask, workers)` вместе с `manager.collectResults(workers)`?

- **A)** Запускает агентов параллельно без координатора
- **B)** Реализует паттерн Hierarchical: Manager Agent распределяет задачи и собирает результаты Worker Agents
- **C)** Реализует консенсусное обсуждение в GroupChat
- **D)** Вызывает одного агента с несколькими промптами по очереди

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Это код паттерна **Hierarchical** из темы 8.1:

```kotlin
val manager = ManagerAgent()
val workers = listOf(workerA, workerB, workerC)
manager.distributeTask(complexTask, workers)
val results = manager.collectResults(workers)
```

Когда использовать: *«Сложные задачи, требующие координации»*. В теме 8.3 этот же принцип развит в `ProjectManagerAgent`: `analyzeAndDistribute(...)` → параллельный `async`/`awaitAll` → `assembleAndValidate(...)`.

**Почему другие не подходят:**
- A — здесь как раз есть координатор (`manager`), значит это не «просто параллельно»
- C — у Collaborative нет distribute/collect, есть `GroupChat.discuss`
- D — sequential подразумевает цепочку `A → B → C`, а не менеджера и воркеров

</details>

---

## Вопрос 5. Из каких четырёх компонентов состоит LangChain согласно модулю?

- **A)** Chains, Agents, Memory, Tools
- **B)** ConversableAgent, GroupChat, UserProxyAgent, GroupChatManager
- **C)** Agent, Task, Crew, Process
- **D)** LLM, Prompt, Token, Context

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Тема 8.2 перечисляет компоненты LangChain:

- **Chains** — последовательность вызовов LLM
- **Agents** — автономные агенты с инструментами
- **Memory** — контекст и история диалога
- **Tools** — внешние функции, доступные агентам

LangChain описан как самый популярный фреймворк для работы с LLM и агентами; для Kotlin есть bindings (langchain4j).

**Почему другие не подходят:**
- B — это концепции AutoGen
- C — это концепции CrewAI
- D — общие термины LLM, в модуле не заявленные как компоненты фреймворка

</details>

---

## Вопрос 6. Какой фреймворк в модуле описан как «conversation-based workflows» и использует ConversableAgent, GroupChat и UserProxyAgent?

- **A)** LangChain
- **B)** AutoGen (Microsoft)
- **C)** CrewAI
- **D)** llama.cpp

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Тема 8.2: **AutoGen** — фреймворк для multi-agent conversation-based workflows. Ключевые концепции:

- **ConversableAgent** — агент, который может общаться с другими агентами
- **GroupChat** — управление группой агентов в обсуждении
- **UserProxyAgent** — агент для взаимодействия с пользователем

В примере `coder` и `reviewer` — `ConversableAgent`, а `user_proxy.initiate_chat(coder, ..., max_turns=10)` запускает диалог.

**Почему другие не подходят:**
- A — у LangChain Chains/Agents/Memory/Tools
- C — CrewAI делает упор на роли (role-based), а не на conversation
- D — llama.cpp это движок инференса из модуля 9, не фреймворк оркестрации

</details>

---

## Вопрос 7. Что такое GroupChat в AutoGen?

- **A)** Просто чат пользователя с одной моделью
- **B)** Управление обсуждением группы агентов (вместе с GroupChatManager)
- **C)** Группировка задач внутри CrewAI
- **D)** Хранилище истории сообщений в LangChain Memory

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Из кода темы 8.2:

```python
groupchat = GroupChat(
    agents=[coder, reviewer, tester],
    messages=[],
    max_round=10,
)
manager = GroupChatManager(groupchat=groupchat, llm_config=...)
```

`GroupChat` задаёт состав участников и лимит раундов, а `GroupChatManager` координирует обсуждение. Это и есть паттерн **Collaborative**: агенты с разной экспертизой приходят к решению в диалоге.

**Почему другие не подходят:**
- A — для общения с пользователем есть `UserProxyAgent`
- C — в CrewAI для группировки есть `Crew`, а не GroupChat
- D — Memory в LangChain хранит контекст/историю, но не управляет обсуждением агентов

</details>

---

## Вопрос 8. Что такое Tool в контексте агентов?

- **A)** Графический интерфейс настройки агента
- **B)** Внешняя функция, доступная агенту (например, `@Tool("Code review tool")`)
- **C)** Просто папка с библиотеками кода в проекте
- **D)** Встроенный терминал для запуска команд

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В модуле Tool встречается в двух видах:

```python
Tool(name="Code Review", func=code_review_function,
     description="Review code for bugs and best practices")
```

```kotlin
@Tool("Code review tool")
fun codeReview(code: String): String { ... }
```

Агент получает набор таких инструментов (`ZeroShotAgent.from_llm_and_tools(llm, tools, ...)`, `.tools(codeReview, generateTests)`) и сам решает, какой из них вызвать.

**Почему другие не подходят:**
- A — GUI не имеет отношения к определению Tool
- C — инструмент != папка с зависимостями; это вызываемая функция
- D — терминал может быть обёрнут в Tool, но это не определение

</details>

---

## Вопрос 9. В CrewAI агенты описаны через `role`, `goal`, `backstory` и `tools`, задачи — через `description` и `expected_output`. Как назвать команду и какой `process` выбрать для линейного выполнения?

- **A)** `Crew` и `Process.SEQUENTIAL`
- **B)** `GroupChat` и `Process.HIERARCHICAL`
- **C)** `Chain` и `Process.PARALLEL`
- **D)** `Agent` и `Process.CONCURRENT`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Тема 8.2 (CrewAI — role-based orchestration): **Agent** — агент с ролью, личностью и инструментами; **Task** — задача с ожидаемым выводом; **Crew** — команда агентов, работающих вместе.

```python
crew = Crew(
    agents=[code_generator, code_reviewer, test_generator],
    tasks=[task1, task2, task3],
    process=Process.SEQUENTIAL,  # или HIERARCHICAL
)
result = crew.kickoff()
```

Для линейного выполнения используют `Process.SEQUENTIAL`; альтернатива из модуля — `HIERARCHICAL`.

**Почему другие не подходят:**
- B — GroupChat это AutoGen, а HIERARCHICAL — не линейный процесс
- C — Chain это LangChain; `Process.PARALLEL` в модуле не описан
- D — Agent — один участник, а не команда; `CONCURRENT` в модуле не встречается

</details>

---

## Вопрос 10. ⚠️ Внимательно прочитай фрагмент промпта Code Review Agent. Какой формат вывода он требует?

```kotlin
val codeReviewPrompt = """
You are a senior Kotlin developer with expertise in:
- Clean Architecture and DDD
- KMP best practices (expect/actual, commonMain organization)
- Security vulnerabilities (SQL injection, XSS, auth issues)

Review the following code and provide:
1. Security issues (critical)
2. Code smells and anti-patterns
3. Performance concerns
4. Testability issues
5. Suggestions for improvement

Code to review:
{code}

Output format: JSON with sections for each category.
"""
```

- **A)** Markdown-отчёт с заголовками для каждой из пяти категорий
- **B)** JSON с секциями для каждой категории
- **C)** Kotlin-файлы тестов с правильной структурой пакетов
- **D)** Свободный текст с замечаниями без структуры

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Это вопрос-ловушка. Промпт действительно перечисляет 5 категорий, и легко решить, что речь идёт о markdown-отчёте. Но последняя строка явно говорит: **«Output format: JSON with sections for each category»** — значит нужен структурированный JSON по категориям.

**Почему другие не подходят:**
- A — ловушка: нумерованный список категорий — это не формат вывода, формат задан отдельной строкой
- C — Kotlin test files — это выходной формат **Test Generator Agent**, а не Code Review Agent
- D — промпт требует именно структурированный JSON, а не свободный текст

</details>

---

## Вопрос 11. Что генерирует Test Generator Agent и какие требования к нему предъявляются?

- **A)** Unit-тесты (JUnit 5, Kotest), интеграционные тесты и Compose UI-тесты; 90%+ покрытие, edge cases, моки через Koin, TestDispatcher
- **B)** pytest-тесты для Python-сервиса оркестрации
- **C)** Только UI-тесты Espresso для Android
- **D)** Тесты на Swift для iOS-модуля

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Ещё одна ловушка: фреймворки оркестрации в модуле Python-based (LangChain, AutoGen, CrewAI), поэтому кажется, что и тесты будут Python-овыми. Но Test Generator Agent работает с **Kotlin-кодом** и его промпт прямо указывает:

1. Unit tests (JUnit 5, Kotest) для business logic
2. Integration tests для Repository и Network layers
3. UI tests (Compose Test) для screens

Требования: 90%+ code coverage, edge cases и негативные сценарии, моки через Koin, `TestDispatcher` для корутин. Output format — Kotlin test files with proper package structure.

**Почему другие не подходят:**
- B — pytest здесь нигде не упоминается; агент тестирует KMP-код
- C — не только UI: есть unit и integration тесты, причём Compose Test, а не Espresso
- D — Swift-тесты не входят в промпт агента

</details>

---

## Вопрос 12. Что проверяет Security Auditor Agent?

- **A)** Только certificate pinning и TLS 1.3
- **B)** Certificate pinning, TLS 1.3, зашифрованное хранилище (SQLCipher, Keystore, Keychain), JWT/OAuth2 и GDPR
- **C)** Только приватность данных и GDPR-соответствие
- **D)** Только скорость генерации кода

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Промпт Security Auditor Agent из темы 8.3 проверяет 5 направлений:

1. Certificate pinning implementation
2. Encrypted storage (SQLCipher, Android Keystore, iOS Keychain)
3. API authentication (JWT, OAuth2)
4. Data privacy (GDPR compliance)
5. Secure network communication (TLS 1.3, pinning)

Формат результата — Security report с уровнями серьёзности **Critical, High, Medium, Low**.

**Почему другие не подходят:**
- A и C — верные, но неполные перечни (ловушка «только …»)
- D — производительность смотрит Code Review Agent (performance concerns), а не security-аудитор

</details>

---

## Вопрос 13. Что происходит на шаге 3 последовательного workflow `CodeDevelopmentWorkflow`?

```kotlin
val finalCode = if (review.hasCriticalIssues) {
    codeGenerator.execute(
        "Fix these issues in the code:\n$review\nOriginal code:\n$code"
    )
} else {
    code
}
val tests = testGenerator.execute("Generate tests for:\n$finalCode")
```

- **A)** Code Review Agent сам исправляет найденные ошибки
- **B)** Code Generator Agent получает отчёт review и оригинальный код для исправления, после чего генерируются тесты
- **C)** Workflow прерывается с ошибкой при любых замечаниях
- **D)** Тесты генерируются сразу для изначального кода, без учёта review

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Шаги `developFeature` из темы 8.3:

1. `codeGenerator.execute("Generate KMP code for: $requirement")`
2. `codeReviewer.execute("Review this code:\n$code")`
3. **Исправление по review (если нужно):** при `review.hasCriticalIssues` исправляет всё тот же `codeGenerator`, получая и review, и оригинальный код
4. `testGenerator.execute("Generate tests for:\n$finalCode")`

Ключевой нюанс: ревьюер только проверяет, исправляет генератор. Тесты всегда строятся для `finalCode`, а не для промежуточного варианта.

**Почему другие не подходят:**
- A — у ревьюера роль анализа, он не модифицирует код
- C — при отсутствии критических проблем используется исходный код (`else code`)
- D — тесты генерируются для `finalCode`, то есть после исправления

</details>

---

## Вопрос 14. Как KMP-приложение запускает Python-оркестратор через REST API в модуле?

- **A)** `POST http://localhost:8000/generate-code`, на стороне сервера FastAPI и `crew.kickoff(...)`
- **B)** `POST http://localhost:11434/api/generate`, сервер Ollama
- **C)** `GET http://localhost:3000/api/agents`, сервер Express
- **D)** `POST http://localhost:5000/review`, сервер Flask

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Тема 8.4, вариант 1 — REST API сервер:

```kotlin
client.post("http://localhost:8000/generate-code") {
    setBody(requirement)
}.body<String>()
```

```python
app = FastAPI()

@app.post("/generate-code")
async def generate_code(requirement: str):
    result = crew.kickoff(inputs={"requirement": requirement})
    return {"code": result.raw}
```

Интерфейс `AIOrchestrationService` объявлен в `commonMain`, а реализация — в `androidMain`/`iosMain`. Вариант 2 — Docker-контейнер с `ai-orchestrator` и пробросом порта `8000:8000`.

**Почему другие не подходят:**
- B — это локальные LLM из модуля 9, а не оркестратор агентов (ловушка на смежный модуль)
- C, D — таких стеков и адресов в модуле нет

</details>

---

## Вопрос 15. Что делает приведённый фрагмент GitHub Actions?

```yaml
name: AI-Powered Code Review
on: [pull_request]

jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - name: Run AI Code Review
        run: |
          python ai_review_workflow.py \
            --code-dir src/commonMain \
            --output reports/ai-review.json
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      - name: Post Review Comment
        uses: actions/github-script@v6
```

- **A)** При каждом PR запускает AI-ревью, сохраняет JSON-отчёт артефактом и публикует его комментарием в PR
- **B)** Запускает ревью только по расписанию раз в неделю
- **C)** Собирает приложение и публикует его в App Store / Google Play
- **D)** Генерирует unit-тесты и автоматически мержит PR

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Ключевые строки конфига:

- `on: [pull_request]` — запуск на каждый PR
- `--code-dir src/commonMain --output reports/ai-review.json` — анализ кода и JSON-отчёт
- `actions/upload-artifact@v3` — сохранение отчёта
- `actions/github-script@v6` + `github.issues.createComment(...)` — публикация комментария с `review.summary`
- API-ключ берётся из GitHub Secrets

Это в точности цель задания 8.3: «Автоматический AI code review на каждый PR», «Результат публикуется как comment в PR».

**Почему другие не подходят:**
- B — триггер `pull_request`, а не `schedule`
- C — в workflow нет шагов сборки и публикации в сторы
- D — workflow генерирует отчёт, а не тесты и не мержит PR (блокировка merge описана лишь как *опциональный* критерий задания, в конфиге её нет)

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 13–15 | 🎉 Отлично! Паттерны, фреймворки и CI/CD интеграция — твои — переходи к Модулю 9 |
| 10–12 | 📖 Хорошо. Перечитай темы 8.2 (фреймворки) и 8.3 (промпты агентов), посмотри вопросы, где ошибся |
| 7–9 | ⚠️ Пройди модуль ещё раз. Сфокусируйся на различиях LangChain / AutoGen / CrewAI и на паттернах взаимодействия |
| < 7 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🔜 Следующий: [Модуль 9 — Локальные LLM модели](../Module_09_Local_LLM/content.md). Там будет про квантование, Ollama/llama.cpp и гибридный подход.

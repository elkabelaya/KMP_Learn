# Модуль 8: Мультиагентность и оркестрация

## 📋 Обзор модуля

**Продолжительность:** 3 недели  
**Сложность:** Advanced  
**Цель:** Освоить проектирование и реализацию мультиагентных систем для автоматизации сложных workflows разработки

---

## 🎯 Цели обучения

После прохождения модуля вы сможете:
- ✅ Проектировать **multi-agent системы** для сложных задач разработки
- ✅ Использовать фреймворки оркестрации (LangChain, AutoGen, CrewAI)
- ✅ Создавать **специализированных агентов** (Code Review, Test Generator, Security Auditor)
- ✅ Интегрировать multi-agent системы в **CI/CD пайплайны**
- ✅ Настроить **коллаборацию агентов** для end-to-end workflows

---

## 📚 Темы модуля

### Тема 8.1: Основы мультиагентных систем (4 часа)

#### Что такое Multi-Agent Systems?

**Определение:** Система из нескольких автономных AI-агентов, которые взаимодействуют друг с другом для решения сложных задач.

**Ключевые концепции:**
- **Автономность** — каждый агент принимает решения самостоятельно
- **Взаимодействие** — агенты общаются через сообщения
- **Координация** — согласование действий для общей цели
- **Специализация** — каждый агент отвечает за свою область

#### Use Cases в разработке ПО:

1. **Code Review Workflow**
   ```
   Developer → Code Generator Agent → Code Review Agent → Test Generator Agent → Final Code
   ```

2. **Full-Stack Development**
   ```
   Product Manager → Backend Agent ↔ Frontend Agent ↔ DevOps Agent → Deployed App
   ```

3. **Testing Pipeline**
   ```
   Code → Unit Test Agent + Integration Test Agent + UI Test Agent → Coverage Report
   ```

#### Паттерны взаимодействия агентов:

##### 1. Sequential (Последовательное выполнение)
```kotlin
// Агент A выполняет задачу → передает результат Агенту B → Агент C финализирует
val resultA = agentA.execute(task)
val resultB = agentB.execute(resultA.output)
val finalResult = agentC.execute(resultB.output)
```

**Когда использовать:** Линейные workflows с четкой последовательностью

##### 2. Parallel (Параллельная обработка)
```kotlin
// Агенты A, B, C работают параллельно над разными частями задачи
val results = coroutineScope {
    val a = async { agentA.execute(taskPart1) }
    val b = async { agentB.execute(taskPart2) }
    val c = async { agentC.execute(taskPart3) }
    listOf(a.await(), b.await(), c.await())
}
```

**Когда использовать:** Независимые задачи, которые можно выполнять одновременно

##### 3. Hierarchical (Иерархия с менеджером)
```kotlin
// Manager Agent распределяет задачи между Worker Agents
val manager = ManagerAgent()
val workers = listOf(workerA, workerB, workerC)

manager.distributeTask(complexTask, workers)
val results = manager.collectResults(workers)
```

**Когда использовать:** Сложные задачи, требующие координации

##### 4. Collaborative (Совместная работа)
```kotlin
// Агенты обсуждают задачу и приходят к консенсусу
val discussion = GroupChat(listOf(agentA, agentB, agentC))
discussion.discuss(task) { result ->
    // Консенсусное решение
}
```

**Когда использовать:** Задачи, требующие разных экспертиз

---

### Тема 8.2: Фреймворки для оркестрации (6 часов)

#### LangChain (Python + Kotlin bindings)

**Что это:** Самый популярный фреймворк для работы с LLM и агентами

**Основные компоненты:**
- **Chains** — последовательность вызовов LLM
- **Agents** — автономные агенты с инструментами
- **Memory** — контекст и история диалога
- **Tools** — внешние функции, доступные агентам

**Пример на Python:**
```python
from langchain.agents import AgentExecutor, Tool, ZeroShotAgent
from langchain.llms import OpenAI

# Определение инструментов
tools = [
    Tool(
        name="Code Review",
        func=code_review_function,
        description="Review code for bugs and best practices"
    ),
    Tool(
        name="Test Generator",
        func=test_generation_function,
        description="Generate unit tests for code"
    )
]

# Создание агента
llm = OpenAI(temperature=0)
prefix = "You are a helpful coding assistant"
agent = ZeroShotAgent.from_llm_and_tools(llm, tools, prefix=prefix)

# Запуск
agent_executor = AgentExecutor.from_agent_and_tools(
    agent=agent, 
    tools=tools,
    verbose=True
)

result = agent_executor.run("Review this Kotlin code and generate tests")
```

**Kotlin bindings (langchain4j):**
```kotlin
import dev.langchain4j.agent.tool.Tool
import dev.langchain4j.model.chat.ChatLanguageModel

@Tool("Code review tool")
fun codeReview(code: String): String {
    // Реализация code review логики
}

val model = ChatLanguageModelBuilder().build()
val agent = Agent.builder()
    .chatLanguageModel(model)
    .tools(codeReview, generateTests)
    .build()

val response = agent.chat("Review this code: $code")
```

---

#### AutoGen (Microsoft)

**Что это:** Фреймворк для multi-agent conversation-based workflows

**Ключевые концепции:**
- **ConversableAgent** — агент, который может общаться с другими агентами
- **GroupChat** — управление группой агентов в обсуждении
- **UserProxyAgent** — агент для взаимодействия с пользователем

**Пример на Python:**
```python
from autogen import ConversableAgent, UserProxyAgent, config_list_from_json

# Загрузка API keys
config_list = config_list_from_json("OAI_CONFIG_LIST")

# Создание агентов
coder = ConversableAgent(
    name="Coder",
    system_message="You are a Kotlin developer. Write clean, tested code.",
    llm_config={"config_list": config_list},
)

reviewer = ConversableAgent(
    name="Reviewer", 
    system_message="You review code for bugs, security issues, and best practices.",
    llm_config={"config_list": config_list},
)

user_proxy = UserProxyAgent(
    name="User",
    code_execution_config=False,
)

# Запуск conversation
user_proxy.initiate_chat(
    recipient=coder,
    message="Create a KMP project with Clean Architecture for habit tracking app",
    max_turns=10,
)
```

**GroupChat пример:**
```python
from autogen import GroupChat, GroupChatManager

# Создание группы агентов
coder = ConversableAgent(name="Coder", ...)
reviewer = ConversableAgent(name="Reviewer", ...)
tester = ConversableAgent(name="Tester", ...)

groupchat = GroupChat(
    agents=[coder, reviewer, tester],
    messages=[],
    max_round=10,
)

manager = GroupChatManager(groupchat=groupchat, llm_config={"config_list": config_list})

# Запуск группового обсуждения
user_proxy.initiate_chat(manager, message="Build a KMP app with tests and documentation")
```

---

#### CrewAI (Role-based orchestration)

**Что это:** Фреймворк для создания команд агентов с ролями

**Основные концепции:**
- **Agent** — агент с ролью, личностью и инструментами
- **Task** — задача для агента с ожидаемым выводом
- **Crew** — команда агентов, работающих вместе

**Пример на Python:**
```python
from crewai import Agent, Task, Crew, Process

# Определение агентов
code_generator = Agent(
    role='Senior Kotlin Developer',
    goal='Generate clean, production-ready KMP code',
    backstory='You are an expert in Kotlin Multiplatform with 10 years experience.',
    tools=[code_generation_tool],
    verbose=True,
)

code_reviewer = Agent(
    role='Code Reviewer',
    goal='Review code for quality, security, and best practices',
    backstory='You are a meticulous reviewer who catches all bugs.',
    tools=[code_review_tool],
    verbose=True,
)

test_generator = Agent(
    role='QA Engineer',
    goal='Generate comprehensive unit and integration tests',
    backstory='You ensure 100% test coverage with edge cases.',
    tools=[test_generation_tool],
    verbose=True,
)

# Определение задач
task1 = Task(
    description='Create a KMP project with Clean Architecture for habit tracking',
    expected_output='Complete Kotlin code with Domain, Data, UI layers',
    agent=code_generator,
)

task2 = Task(
    description='Review the generated code for bugs and best practices',
    expected_output='Code review report with suggestions',
    agent=code_reviewer,
)

task3 = Task(
    description='Generate unit tests for all business logic',
    expected_output='JUnit 5 test files with 90%+ coverage',
    agent=test_generator,
)

# Создание команды
crew = Crew(
    agents=[code_generator, code_reviewer, test_generator],
    tasks=[task1, task2, task3],
    process=Process.SEQUENTIAL,  # или HIERARCHICAL
    verbose=True,
)

# Запуск
result = crew.kickoff()
print(result)
```

---

### Тема 8.3: Практическая оркестрация в KMP проекте (8 часов)

#### Создание специализированных агентов для разработки

##### 1. Code Review Agent

**Роль:** Анализ сгенерированного кода на качество и уязвимости

**Промпт:**
```kotlin
val codeReviewPrompt = """
You are a senior Kotlin developer with expertise in:
- Clean Architecture and DDD
- KMP best practices (expect/actual, commonMain organization)
- Security vulnerabilities (SQL injection, XSS, auth issues)
- Performance anti-patterns

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

**Инструменты:**
- Статический анализ (Detekt, Ktlint)
- Security сканеры (OWASP ZAP integration)
- Code complexity metrics

##### 2. Test Generator Agent

**Роль:** Создание unit, integration и UI тестов

**Промпт:**
```kotlin
val testGenerationPrompt = """
You are a QA engineer specializing in Kotlin testing.

Generate comprehensive tests for:
1. Unit tests (JUnit 5, Kotest) для business logic
2. Integration tests для Repository и Network layers
3. UI tests (Compose Test) для screens

Requirements:
- 90%+ code coverage
- Edge cases и негативные сценарии
- Mock объектов через Koin
- Coroutines TestDispatcher для асинхронности

Code to test:
{code}

Output format: Kotlin test files with proper package structure.
"""
```

##### 3. Documentation Agent

**Роль:** Генерация KDoc, README и onboarding guides

**Промпт:**
```kotlin
val documentationPrompt = """
You are a technical writer for Kotlin projects.

Generate:
1. KDoc комментарии для всех public API
2. README.md с установкой, запуском, архитектурой
3. Onboarding guide для новых разработчиков
4. ADR (Architecture Decision Records) для ключевых решений

Project context:
{projectStructure}

Code to document:
{code}

Output format: Markdown files with proper formatting.
"""
```

##### 4. Security Auditor Agent

**Роль:** Проверка на уязвимости безопасности

**Промпт:**
```kotlin
val securityAuditPrompt = """
You are a security engineer specializing in mobile app security.

Check for:
1. Certificate pinning implementation
2. Encrypted storage (SQLCipher, Android Keystore, iOS Keychain)
3. API authentication (JWT, OAuth2)
4. Data privacy (GDPR compliance)
5. Secure network communication (TLS 1.3, pinning)

Code to audit:
{code}

Output format: Security report with severity levels (Critical, High, Medium, Low).
"""
```

---

#### Интеграция агентов в workflow разработки

##### Sequential Workflow (Code → Review → Test)
```kotlin
class CodeDevelopmentWorkflow(
    private val codeGenerator: Agent,
    private val codeReviewer: Agent,
    private val testGenerator: Agent,
) {
    suspend fun developFeature(requirement: String): DevelopmentResult {
        // Шаг 1: Генерация кода
        val code = codeGenerator.execute(
            "Generate KMP code for: $requirement"
        )
        
        // Шаг 2: Code review
        val review = codeReviewer.execute(
            "Review this code:\n$code"
        )
        
        // Шаг 3: Исправление по review (если нужно)
        val finalCode = if (review.hasCriticalIssues) {
            codeGenerator.execute(
                "Fix these issues in the code:\n$review\nOriginal code:\n$code"
            )
        } else {
            code
        }
        
        // Шаг 4: Генерация тестов
        val tests = testGenerator.execute(
            "Generate tests for:\n$finalCode"
        )
        
        return DevelopmentResult(
            code = finalCode,
            review = review,
            tests = tests
        )
    }
}
```

##### Parallel Workflow (Multiple Features)
```kotlin
class ParallelFeatureDevelopment(
    private val agents: List<CodeGeneratorAgent>,
) {
    suspend fun developFeatures(features: List<String>): Map<String, String> {
        return coroutineScope {
            features.map { feature ->
                async {
                    val agent = agents[features.indexOf(feature)]
                    Pair(feature, agent.execute("Implement: $feature"))
                }
            }.awaitAll().toMap()
        }
    }
}
```

##### Hierarchical Workflow (Manager + Workers)
```kotlin
class ProjectManagerAgent(
    private val workers: List<Agent>,
) {
    suspend fun manageProject(projectSpec: ProjectSpecification): ProjectResult {
        // Анализ задачи и распределение между workers
        val taskAssignments = analyzeAndDistribute(projectSpec, workers)
        
        // Параллельное выполнение
        val results = coroutineScope {
            taskAssignments.map { (agent, task) ->
                async { agent.execute(task) }
            }.awaitAll()
        }
        
        // Сборка и валидация результатов
        val finalResult = assembleAndValidate(results, projectSpec)
        
        return finalResult
    }
}
```

---

### Тема 8.4: Интеграция с Kotlin и CI/CD (6 часов)

#### Запуск Python фреймворков из KMP проекта

**Вариант 1: REST API сервер**
```kotlin
// commonMain
interface AIOrchestrationService {
    suspend fun generateCode(requirement: String): String
    suspend fun reviewCode(code: String): CodeReview
    suspend fun generateTests(code: String): List<TestFile>
}

// androidMain / iosMain
class AIOrchestrationServiceImpl : AIOrchestrationService {
    private val client = KtorClient()
    
    override suspend fun generateCode(requirement: String): String {
        return client.post("http://localhost:8000/generate-code") {
            setBody(requirement)
        }.body<String>()
    }
}

// Python сервер (FastAPI)
from fastapi import FastAPI
from crewai import Crew

app = FastAPI()

@app.post("/generate-code")
async def generate_code(requirement: str):
    result = crew.kickoff(inputs={"requirement": requirement})
    return {"code": result.raw}
```

**Вариант 2: Docker контейнер с orchestration сервисом**
```yaml
# docker-compose.yml
version: '3.8'
services:
  ai-orchestrator:
    build: ./ai-orchestration-service
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - CLAUDE_API_KEY=${CLAUDE_API_KEY}
    volumes:
      - ./models:/app/models
```

#### Интеграция в CI/CD пайплайн

**GitHub Actions пример:**
```yaml
name: AI-Powered Code Review

on: [pull_request]

jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install AI orchestration tools
        run: |
          pip install crewai langchain autogen
      
      - name: Run AI Code Review
        run: |
          python ai_review_workflow.py \
            --code-dir src/commonMain \
            --output reports/ai-review.json
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      
      - name: Upload Review Report
        uses: actions/upload-artifact@v3
        with:
          name: ai-code-review
          path: reports/ai-review.json
      
      - name: Post Review Comment
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const review = JSON.parse(fs.readFileSync('reports/ai-review.json'));
            github.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## AI Code Review\n\n${review.summary}`
            });
```

---

## 📝 Практические задания

### Задание 8.1: Создать Crew из 3 агентов для code review workflow (4 часа)

**Цель:** Настроить CrewAI с Code Generator, Reviewer и Test Generator агентами

**Шаги:**
1. Установить Python 3.11+ и CrewAI
2. Создать 3 агента с разными ролями
3. Определить sequential процесс выполнения
4. Протестировать на реальном KMP коде из EcoTrack

**Критерии успеха:**
- ✓ Все 3 агента работают корректно
- ✓ Workflow выполняется end-to-end
- ✓ Генерируемый код проходит review и имеет тесты

---

### Задание 8.2: Настроить AutoGen conversation для генерации + тестирование кода (4 часа)

**Цель:** Создать GroupChat с Coder, Reviewer и Tester агентами

**Шаги:**
1. Настроить AutoGen с Claude API
2. Создать 3 ConversableAgent с разными system_message
3. Настроить GroupChatManager для координации
4. Запустить conversation с requirement

**Критерии успеха:**
- ✓ Агенты общаются друг с другом (не только с user)
- ✓ Conversation завершается рабочим кодом + тестами
- ✓ Максимум 10 rounds обсуждения

---

### Задание 8.3: Интегрировать multi-agent систему в CI/CD пайплайн (6 часов)

**Цель:** Автоматический AI code review на каждый PR

**Шаги:**
1. Создать Python сервис с FastAPI для orchestration
2. Написать GitHub Actions workflow
3. Интегрировать с существующим KMP проектом
4. Настроить API keys в GitHub Secrets

**Критерии успеха:**
- ✓ AI review запускается автоматически на PR
- ✓ Результат публикуется как comment в PR
- ✓ Критические issues блокируют merge (опционально)

---

## 📊 Проверка знаний

### Quiz (10 вопросов)

1. **Какой паттерн лучше для независимых задач?**
   - a) Sequential
   - b) Parallel ✓
   - c) Hierarchical

2. **Что такое GroupChat в AutoGen?**
   - a) Чат с пользователем
   - b) Управление обсуждением группы агентов ✓
   - c) Группировка задач

3. **Какой фреймворк использует role-based подход?**
   - a) LangChain
   - b) AutoGen
   - c) CrewAI ✓

4. **Что такое Tool в контексте агентов?**
   - a) GUI инструмент
   - b) Внешняя функция, доступная агенту ✓
   - c) Библиотека кода

5. **Какой процесс в CrewAI для линейного выполнения?**
   - a) SEQUENTIAL ✓
   - b) HIERARCHICAL
   - c) CONCURRENT

(и ещё 5 вопросов...)

---

## 🎯 Итоговый проект модуля

**Задача:** Создать AI-Powered Development Workflow для KMP проекта

**Требования:**
1. Multi-agent система с минимум 3 агентами (Generator, Reviewer, Tester)
2. Интеграция с CI/CD (автоматический review на PR)
3. REST API для ручного запуска workflows
4. Документация по настройке и использованию

**Критерии оценки:**
- ✓ Система работает end-to-end
- ✓ Уменьшает время code review на 50%+
- ✓ Генерирует тесты с 80%+ coverage

---

## 📚 Дополнительные материалы

### Документация:
- [LangChain Documentation](https://python.langchain.com/)
- [AutoGen Documentation](https://microsoft.github.io/autogen/)
- [CrewAI Documentation](https://docs.crewai.com/)

### Примеры:
- [CrewAI Examples](https://github.com/joaomdmoura/crewai-examples)
- [AutoGen Samples](https://github.com/microsoft/autogen/tree/main/samples)

### Видео:
- "Building Multi-Agent Systems with CrewAI" (YouTube, 45 мин)
- "AutoGen Deep Dive" (Microsoft, 60 мин)

---

## 🚀 Следующий шаг

Переходите к [Модулю 9](../Module_09_Local_LLM/content.md): Локальные LLM модели

**Время до следующего модуля:** 3 недели  
**Рекомендуемая практика:** Ежедневно запускать multi-agent workflows на реальных задачах

---

**Удачи в освоении мультиагентных систем! 🤖🤖🤖**

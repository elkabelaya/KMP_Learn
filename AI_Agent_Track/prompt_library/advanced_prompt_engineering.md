# Advanced Prompt Engineering для KMP

## 🎯 Принципы эффективных prompt'ов

### 1. Контекст и Роль (Context & Role)

**❌ Плохо:**
```
Создай repository для User.
```

**✅ Хорошо:**
```
Ты - Senior Kotlin Multiplatform разработчик с 5+ лет опытом в Clean Architecture.

Контекст проекта:
- E-commerce приложение с 1M+ пользователей  
- Feature-first modularization (feature-user модуль)
- Offline-first с SQLDelight persistence  
- Koin для DI, Result sealed class для error handling

Задача:
Создай UserRepository implementation...
```

**Почему работает:** AI понимает ваш контекст и использует appropriate patterns.

---

### 2. Конкретные Требования (Specific Requirements)

**❌ Плохо:**
```
Сделай хороший код.
```

**✅ Хорошо:**
```
Требования к коду:
1. Return Result<T, Error> sealed class (не throw exceptions)
2. Use coroutines с proper dispatcher (IO для DB, Default для business logic)  
3. Follow Single Responsibility Principle (один класс - одна ответственность)
4. Include KDoc documentation для всех public API
5. Support cancellation через Job parameter

Constraints:
- Kotlin 1.9+ syntax  
- No platform-specific code в commonMain
- SQLDelight 2.0+ API
```

**Почему работает:** Четкие требования = предсказуемый результат.

---

### 3. Формат Вывода (Output Format)

**❌ Плохо:**
```
Покажи код.
```

**✅ Хорошо:**
```
Формат вывода:
1. Сначала объясни architectural decisions (2-3 paragraphs)
2. Покажи полную структуру файлов с tree view  
3. Для каждого файла:
   - File path
   - Полный код с proper formatting
   - KDoc комментарии включены
4. В конце покажи example usage code

Используй Markdown code blocks с language specification:
```kotlin
// Code here
```
```

**Почему работает:** Вы контролируете структуру ответа.

---

### 4. Примеры и Few-Shot Learning

**❌ Плохо:**
```
Создай data class для Product.
```

**✅ Хорошо:**
```
Создай data class для Product по аналогии с User:

Пример (User):
```kotlin
@Serializable
data class User(
    val id: UserId,
    val email: Email,  
    val name: Name,
    val createdAt: Instant
) {
    companion object {
        fun create(email: String, name: String): Result<User, ValidationError> {
            // Validation logic...
        }
    }
}
```

Создай аналогично для Product с полями:
- id: ProductId  
- name: String (1-200 chars)
- price: Money
- description: String?  
- category: CategoryId
- createdAt: Instant

Включи validation в companion object.
```

**Почему работает:** AI лучше понимает паттерн через примеры.

---

### 5. Iterative Refinement (Итеративное Улучшение)

**Первый prompt:**
```
Создай API client для User endpoints.
```

**AI генерирует код...**

**Follow-up prompt:**
```
Хорошо, но добавь:
1. Retry policy с exponential backoff (max 3 retries)  
2. Auto token refresh на 401 Unauthorized
3. Request/response logging interceptor для debug builds

Обнови предыдущий код с этими изменениями.
```

**Почему работает:** Постепенное уточшение лучше чем один сложный prompt.

---

## 🚀 Advanced Prompt Patterns

### Pattern 1: Chain of Thought (CoT)

```
Ты - Senior Architect. Спроектируй authentication flow для KMP приложения.

Думай step-by-step:
1. Сначала проанализируй требования и constraints  
2. Предложи 2-3 архитектурных подхода с pros/cons каждого
3. Выбери лучший подход и обоснуй выбор  
4. Покажи detailed design с component diagram
5. Реализуй код для выбранного дизайна

Начни с анализа требований...
```

**Когда использовать:** Для complex architectural decisions.

---

### Pattern 2: Self-Correction

```
Ты - Senior Kotlin разработчик.

Задача: Создай thread-safe cache implementation для KMP.

Перед написанием кода:
1. Сгенерируй initial solution  
2. Critique свой код - найди potential issues (race conditions, memory leaks, etc.)
3. Refactor код чтобы fix найденные issues  
4. Покажи final version с объяснением изменений

Начни с initial solution...
```

**Когда использовать:** Для critical code где quality важнее скорости.

---

### Pattern 3: Multiple Perspectives

```
Ты - Panel of Experts reviewing этот код:
- Senior Kotlin Developer (focus на idiomatic Kotlin)  
- Security Engineer (focus на vulnerabilities)
- Performance Engineer (focus на efficiency)  
- QA Engineer (focus on testability)

Код для review:
[ВСТАВИТЬ КОД]

Каждый expert должен:
1. Identify issues из своей perspective  
2. Rate severity (Critical/High/Medium/Low)
3. Suggest specific fixes

После всех reviews, создай consolidated action plan prioritized по severity.
```

**Когда использовать:** Для comprehensive code review перед production.

---

### Pattern 4: Constraint-Based Generation

```
Создай solution с жесткими constraints:

Functional Requirements:
- Must support offline mode  
- Must sync when online again
- Must handle conflicts с "last write wins" strategy

Non-Functional Requirements:  
- Memory usage < 50MB additional
- Sync latency < 2 seconds для 100 items  
- Must work на Android API 21+ и iOS 13+

Constraints:
- NO external libraries кроме SQLDelight  
- NO platform-specific code в commonMain
- Must compile с Kotlin 1.9

Покажи solution что удовлетворяет ВСЕМ requirements и constraints.
Если constraint невозможно satisfy, объясни почему и предложи alternative.
```

**Когда использовать:** Для production code с strict requirements.

---

### Pattern 5: Test-Driven Development

```
Ты - Senior разработчик практикующий TDD.

Задача: Implement [FEATURE] с test-driven approach.

Process:
1. Сначала напиши failing test для primary use case  
2. Покажи минимальный implementation чтобы test passed
3. Напиши additional tests для edge cases (один за раз)  
4. Для каждого failing test, покажи minimal implementation
5. После всех tests passing, refactor code (без изменения behavior)

Начни с первого failing test...
```

**Когда использовать:** Для learning TDD или critical business logic.

---

## 🎨 Prompt Templates для Common Scenarios

### Template 1: Code Refactoring

```
Ты - Senior Kotlin разработчик, эксперт в code refactoring.

Задача: Refactor следующий код чтобы улучшить [GOAL - maintainability/performance/security].

Current Code:
[ВСТАВИТЬ КОД]

Refactoring Goals (prioritized):
1. [Primary goal - например: Extract use cases из ViewModel]  
2. [Secondary goal - например: Add proper error handling]
3. [Tertiary goal - например: Improve naming и documentation]

Constraints:
- Maintain backward compatibility (public API не должен измениться)  
- All existing tests должны continue passing
- No breaking changes для consumers

Process:
1. Analyze current code и identify smells/issues  
2. Propose refactoring plan с steps
3. Show refactored code step-by-step  
4. Explain benefits каждого change

Начни с analysis...
```

---

### Template 2: Debugging Assistance

```
Ты - Senior Kotlin разработчик, эксперт в debugging.

Проблема: [ОПИСАНИЕ ПРОБЛЕМЫ - что происходит vs что должно]

Контекст:
- Когда воспроизводится: [conditions]  
- Frequency: [always/sometimes/rarely]
- Error message или stack trace (если есть): [ERROR]

Relevant Code:
[ВСТАВИТЬ КОД]

Debugging Steps Taken:
- [Step 1 - что уже пробовали]  
- [Step 2 - результаты]

Задача:
1. Analyze проблему и предложи root cause hypotheses (top 3)  
2. Для каждой hypothesis покажи how to verify
3. Предложи fix для наиболее вероятной root cause  
4. Покажи как prevent эту проблему в будущем

Начни с analysis...
```

---

### Template 3: Performance Optimization

```
Ты - Performance Engineer с опытом в mobile applications.

Задача: Optimize performance для [COMPONENT/FEATURE].

Current Performance Metrics:
- Cold start time: [X] seconds (target: <[Y])  
- Memory usage: [X] MB (target: <[Y])
- Frame drops: [X]% (target: <[Y]%)  
- Network latency: [X] ms (target: <[Y])

Relevant Code:
[ВСТАВИТЬ КОД]

Profiling Data (если есть):
- CPU hotspots: [DATA]  
- Memory allocations: [DATA]
- Network requests: [DATA]

Задача:
1. Identify performance bottlenecks из code и profiling data  
2. Rank по impact (high/medium/low)
3. Для каждого bottleneck покажи:
   - Why это bottleneck  
   - How to fix с конкретным кодом
   - Expected improvement

Предложи optimizations prioritized по impact/effort ratio.
```

---

## ⚠️ Anti-Patterns и Ошибки

### ❌ Too Vague
```
Сделай что-то для authentication.
```

**Проблема:** AI не понимает scope, requirements, constraints.  
**Решение:** Будьте specific - что именно нужно?

---

### ❌ Too Complex в одном prompt
```
Создай полное e-commerce приложение с authentication, products cart, checkout, payments, analytics, push notifications...
```

**Проблема:** AI перегружен, качество падает.  
**Решение:** Разбей на отдельные prompt'ы по модулям.

---

### ❌ Ignoring AI Limitations
```
Создай production-ready code без testing.
```

**Проблема:** AI может hallucinate, не все edge cases covered.  
**Решение:** Всегда review и test AI-generated code.

---

### ❌ Copy-Paste без Understanding
```
[Использует AI code без понимания что он делает]
```

**Проблема:** Не можете debug или extend позже.  
**Решение:** Поймите каждый line перед использованием.

---

## 📊 Prompt Engineering Best Practices Summary

### ✅ DO:
- **Provide context:** Project type, tech stack, constraints  
- **Be specific:** Clear requirements, expected output format
- **Use examples:** Show what good looks like  
- **Iterate:** Refine prompt based на AI response
- **Review output:** Never deploy без human review

### ❌ DON'T:
- **Be vague:** "Make it work" не помогает  
- **Overload:** Один prompt для всего проекта
- **Trust blindly:** AI может ошибаться  
- **Skip testing:** Always test AI-generated code
- **Forget security:** Security-critical code требует extra review

---

## 🎓 Continuous Improvement

### Track Your Prompts
Ведите log эффективных prompt'ов:
```markdown
## Prompt Library

### [Use Case Name]
**Prompt:** [Your prompt text]  
**Result Quality:** ⭐⭐⭐⭐⭐  
**AI Tool:** [Cursor/Copilot/Claude]  
**Notes:** [What worked well, what to improve]
```

### Share с Team
- Создайте shared prompt library в вашем org  
- Conduct prompt engineering workshops  
- Review и improve prompts регулярно

### Stay Updated
- AI capabilities evolve быстро - обновляйте prompt'ы  
- Follow AI tool release notes для new features  
- Experiment с different models и compare results

---

**Master prompt engineering = Master AI-assisted development! 🎯🤖**

*Помните: Хороший prompt экономит часы работы. Инвестируйте время в crafting good prompts.*

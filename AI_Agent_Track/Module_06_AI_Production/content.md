# Модуль 6: AI в Production

## 📋 Обзор модуля

**Продолжительность:** 8 часов  
**Сложность:** Advanced  
**Цель:** Безопасное использование AI в production, code review с AI, документация и best practices

---

## 🎯 Цели обучения

После прохождения модуля вы сможете:
- ✅ Проводить **AI-assisted code review** для безопасности и качества
- ✅ Генерировать **документацию** автоматически с AI
- ✅ Настроить **AI guardrails** и security checks
- ✅ Создать **team AI guidelines** для production использования

---

## 📚 Темы модуля

### Тема 6.1: AI-assisted Code Review (2 часа)

#### Шаблон prompt'а для code review:

```
Ты - Senior Principal Engineer с 10+ лет опытом, эксперт в security и code quality.

Задача:
Проведи comprehensive code review для следующего кода.

[ВСТАВИТЬ КОД]

Критерии ревью:
1. **Security Issues** (критично)
   - SQL injection, XSS, insecure data storage
   - Hardcoded secrets или API keys
   - Insecure network communication
   - Information leakage

2. **Performance Issues** (высокий приоритет)
   - Memory leaks, unnecessary allocations
   - N+1 queries, inefficient algorithms  
   - Blocking main thread operations

3. **Code Quality** (средний приоритет)
   - Violation of SOLID principles
   - Code duplication, god classes/functions
   - Poor naming, lack of documentation

4. **Testing Coverage** (средний приоритет)
   - Missing error handling tests
   - Uncovered edge cases
   - Flaky test patterns

5. **Kotlin Best Practices** (низкий приоритет)
   - Non-idiomatic Kotlin patterns
   - Missing null safety annotations
   - Improper coroutine usage

Формат вывода:
## 🔴 Critical Issues (Security)
- [Issue description]
  - File: path/to/file.kt, Line: X
  - Problem: [объяснение]
  - Fix: [конкретный код исправления]

## 🟡 High Priority Issues (Performance)
...

## 🟢 Recommendations (Best Practices)
...

## Summary
- Total issues found: X critical, Y high, Z medium
- Security score: /10
- Recommended actions

Будь строгим и конкретным. Покажи код исправлений для каждого issue.
```

---

#### Пример code review с AI:

**Промпт:**
```
Проведи security и quality review для этого repository implementation:

```kotlin
class UserRepository(
    private val apiService: ApiService,
    private val db: Database
) {
    suspend fun getUser(userId: String): User? {
        // Check cache first
        val cached = db.users.get(userId)
        if (cached != null) return cached
        
        // Fetch from API  
        val response = apiService.getUser(userId)
        if (response.success) {
            val user = response.data
            db.users.insert(user) // Potential race condition
            return user
        }
        return null
    }
    
    suspend fun updateUser(userId: String, data: Map<String, Any>) {
        // Direct SQL query - potential injection!
        val query = "UPDATE users SET ${data.keys.joinToString(",") { "$it" }} = ? WHERE id = '$userId'"
        db.execute(query, *data.values.toTypedArray())
    }
}
```

Найди все security и quality issues. Покажи исправленный код.
```

**Ожидаемый результат от AI:**

```markdown
## 🔴 Critical Issues (Security)

### 1. SQL Injection Vulnerability
- **File:** UserRepository.kt, Line: updateUser()
- **Problem:** Direct string interpolation в SQL query позволяет injection attacks
  ```kotlin
  // VULNERABLE:
  "WHERE id = '$userId'"  // Атакующий может передать: "admin' OR '1'='1"
  ```
- **Fix:** Используй parameterized queries через SQLDelight
  ```kotlin
  // SECURE:
  db.users.updateUser(userId, data) // SQLDelight автоматически escape параметры
  ```

### 2. Race Condition в Cache Update
- **File:** UserRepository.kt, Line: db.users.insert(user)
- **Problem:** Между get() и insert() другая корутина может вставить тот же record
- **Fix:** Используй upsert или transaction
  ```kotlin
  db.transaction {
      val user = apiService.getUser(userId)
      db.users.upsert(user) // Atomic operation
  }
  ```

## 🟡 High Priority Issues (Performance)

### 3. Cache Stampede
- **Problem:** При cache miss все запросы идут к API одновременно
- **Fix:** Implement cache stampede protection с mutex или distributed lock

## 🟢 Recommendations

### 4. Missing Error Handling
- **Problem:** updateUser() не обрабатывает DB errors
- **Fix:** Возвращай Result<Unit> вместо void

### 5. Inconsistent Naming
- **Problem:** Смешанный стиль (camelCase и PascalCase)
- **Fix:** Используй consistent naming convention

## Summary
- Total issues: 2 critical, 1 high, 2 medium
- Security score: 3/10 (критичные уязвимости!)
- **Action Required:** Fix SQL injection перед deployment!
```

---

### Тема 6.2: Генерация документации с AI (2 часа)

#### Шаблон prompt'а для API documentation:

```
Ты - Technical Writer с опытом в developer documentation.

Задача:
Создай comprehensive API documentation для следующего кода.

[ВСТАВИТЬ КОД]

Формат документации:
# [Название API/Компонента]

## Overview
[Краткое описание назначения и использования - 2-3 предложения]

## Installation  
[Как добавить dependency или setup]

## Quick Start
```kotlin
// Минимальный пример использования
```

## API Reference

### [ClassName/FunctionName]
**Signature:**
```kotlin
[полная сигнатура с типами]
```

**Description:**
[Что делает, когда использовать]

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| param1 | Type1 | Yes | Описание |

**Returns:**
[Возвращаемое значение и его смысл]

**Throws:**
- [Exception1] - когда бросается
- [Exception2] - когда бросается

**Example:**
```kotlin
// Полный пример использования с error handling
```

## Best Practices
- [Рекомендация 1]
- [Рекомендация 2]

## Common Pitfalls
- [Частая ошибка 1 и как избежать]
- [Частая ошибка 2 и как избежать]

## Related
- [Ссылки на связанные API или компоненты]

Создай документацию для всех public классов и функций.
```

---

#### Шаблон prompt'а для README:

```
Ты - Senior Developer и Technical Writer.

Задача:
Создай профессиональный README.md для KMP проекта.

Контекст проекта:
- Название: [PROJECT NAME]
- Описание: [PROJECT DESCRIPTION]  
- Ключевые фичи: [FEATURE LIST]
- Tech stack: KMP, Compose Multiplatform, SQLDelight, Koin

Структура README:
1. Project badge (version, license, platforms)
2. Project description и screenshots/gif
3. Key features (bullet points)
4. Requirements (Kotlin version, Android/iOS versions)
5. Installation steps (step-by-step)
6. Quick start example code
7. Architecture overview с diagram
8. Module structure explanation  
9. Contributing guidelines
10. License

Создай engaging и informative README в Markdown формате.
```

---

### Тема 6.3: AI Guardrails и Security (2 часа)

#### Шаблон prompt'а для security audit:

```
Ты - Security Engineer, специалист в mobile application security.

Задача:
Проведи comprehensive security audit для KMP приложения.

Анализируй следующие аспекты:

## 1. Data Security
- Encryption at rest (database, shared preferences)
- Encryption in transit (TLS, certificate pinning)
- Sensitive data storage (API keys, tokens, PII)

## 2. Authentication & Authorization  
- Token storage и rotation
- Session management
- Permission escalation vulnerabilities

## 3. Network Security
- API endpoint security  
- Input validation и sanitization
- Rate limiting и abuse prevention

## 4. Code Security
- Obfuscation (R8, ProGuard)
- Debug code в production builds
- Logging sensitive information

## 5. Platform-Specific Security
- Android: Backup prevention, flagSecure, etc.
- iOS: Keychain usage, sandboxing

Для каждого аспекта:
1. Проверь текущую реализацию
2. Выяви уязвимости и риски
3. Предложи конкретные исправления с кодом

[ВСТАВИТЬ КОД ИЛИ ОПИСАНИЕ АРХИТЕКТУРЫ]

Создай security audit report с prioritized action items.
```

---

#### AI Guardrails для production:

**Правила использования AI в production:**

```markdown
# AI Usage Guidelines для Production Code

## ✅ DO:
- Использовать AI для boilerplate и repetitive code
- Генерировать тесты и документацию  
- Получать объяснения сложных концепций
- Рефакторинг с AI assistance

## ⚠️ REVIEW REQUIRED:
- Security-critical code (auth, encryption, payments)
- Business logic с financial implications  
- Data access и persistence layer
- Network communication code

## ❌ DON'T:
- Deploy AI-generated code без human review
- Использовать AI для security implementations без audit
- Доверять AI 100% - всегда тестируйте
- Использовать AI для генерации secrets или credentials

## Security Checklist перед deployment:
- [ ] Все API keys и secrets в environment variables
- [ ] TLS certificate pinning configured  
- [ ] Database encryption enabled
- [ ] No debug code или logging sensitive data
- [ ] Code obfuscation enabled для production builds
- [ ] Security audit completed и все critical issues fixed
```

---

### Тема 6.4: Team AI Guidelines (2 часа)

#### Шаблон team guidelines документа:

```markdown
# AI-Assisted Development Guidelines для [Team Name]

## 🎯 Цель документа
Установить best practices для безопасного и эффективного использования AI в разработке.

## 📋 Policy Overview

### Разрешено:
- ✅ Генерация boilerplate кода (data classes, DTOs, mappers)
- ✅ Создание unit/integration тестов  
- ✅ Генерация документации (KDoc, README)
- ✅ Code review assistance и refactoring suggestions
- ✅ Объяснение сложных концепций и debugging

### Требует approval:
- ⚠️ Security-related implementations (auth, encryption)
- ⚠️ Payment processing и financial logic  
- ⚠️ Core business algorithms
- ⚠️ Production database migrations

### Запрещено:
- ❌ Deployment AI-generated code без human review
- ❌ Использование AI для генерации credentials/secrets
- ❌ Копирование proprietary code через AI tools  
- ❌ Обход security controls с помощью AI

## 🔍 Code Review Process для AI-Generated Code

### Level 1 - Low Risk (Self-review):
- Boilerplate code, data classes
- Unit tests, documentation  
- Simple UI components

**Процесс:** Self-review + peer review (1 person)

### Level 2 - Medium Risk (Team review):
- Business logic, use cases  
- Repository implementations
- Network layer code

**Процесс:** Self-review + peer review (2 people) + security checklist

### Level 3 - High Risk (Architecture review):
- Security implementations  
- Payment processing
- Data access patterns

**Процесс:** Self-review + peer review (2 people) + security audit + architecture review

## 📊 Quality Metrics

### Для AI-generated code:
- Test coverage: ≥80%
- Code review approval: 100%
- Security audit: Passed для Level 2+ code
- Performance benchmarks: Meet SLA requirements

## 🛠️ Approved AI Tools

### Primary Tools:
- GitHub Copilot (enterprise license)
- Cursor.sh (team license)

### Secondary Tools:
- Claude 3.5 Sonnet (для complex tasks)
- Continue.dev (для local models)

### Not Approved:
- Free tier AI tools для production code
- Unknown или unvetted AI services

## 📝 Prompt Engineering Standards

### Хороший prompt включает:
1. **Роль:** "Ты - Senior Kotlin разработчик..."
2. **Контекст:** Описание проекта и архитектуры  
3. **Задача:** Четкое что нужно создать
4. **Ограничения:** Tech stack, versions, constraints
5. **Формат:** Как должен выглядеть вывод

### Пример хорошего prompt:
```
Ты - Senior KMP разработчик. Создай repository для User entity с:
- CRUD operations через Result sealed class
- SQLDelight для persistence  
- In-memory caching с 5min TTL
- Flow для reactive updates

Используй Kotlin 1.9+, следуй Clean Architecture.
Покажи полный код с KDoc комментариями.
```

## 🚨 Incident Response

Если AI сгенерировал problematic code:
1. **Не deploy** в production
2. **Report** в #ai-incidents Slack channel  
3. **Analyze** root cause (bad prompt? AI hallucination?)
4. **Update guidelines** чтобы предотвратить повторение

## 📚 Additional Resources

- [Prompt Library](./prompt-library.md) - Approved prompts
- [Security Checklist](./security-checklist.md) - Pre-deployment checks  
- [Code Review Guidelines](./code-review.md) - Review process

---
**Версия:** 1.0  
**Дата обновления:** 2025-01-XX  
**Owner:** Engineering Lead
```

---

## 📝 Практические задания модуля

### Задание 6.1: AI Code Review (2 часа)

**Требования:**
- Проведите security и quality review для 2-3 файлов из EcoTrack с AI
- Исправьте все critical и high priority issues

**Критерии:**
- ✓ Все security issues исправлены
- ✓ Code quality улучшена

---

### Задание 6.2: Генерация документации (2 часа)

**Требования:**
- Создайте API documentation для 3-5 public классов
- Обновите README.md с AI assistance

**Критерии:**
- ✓ Документация comprehensive и понятная
- ✓ Включает примеры использования

---

### Задание 6.3: Security Audit (2 часа)

**Требования:**
- Проведите security audit всего проекта с AI
- Исправьте все уязвимости

**Критерии:**
- ✓ Security score ≥8/10
- ✓ Все critical issues resolved

---

### Задание 6.4: Team AI Guidelines (2 часа)

**Требования:**
- Создайте team guidelines документ для вашего проекта/команды
- Включите policy, review process, approved tools

**Критерии:**
- ✓ Comprehensive guidelines document
- ✓ Approved team lead

---

## 🚫 Ошибки при использовании AI в production

### ❌ Deployment без testing
**Решение:** Всегда пишите тесты для AI-generated code перед deployment

### ❌ Ignoring security warnings
**Решение:** Treat все security issues как critical - fix перед deployment

### ❌ Over-reliance на AI
**Решение:** Поддерживайте human expertise - AI это assistant, не replacement

---

## 📚 Дополнительные материалы

### Security Resources:
- [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)
- [Android Security Best Practices](https://developer.android.com/topic/security)
- [iOS App Security](https://developer.apple.com/documentation/xcodekit/securingyourapp)

### Documentation Tools:
- [Dokka](https://square.github.io/kotlin-dokka/) - Kotlin documentation generator
- [Swagger/OpenAPI](https://swagger.io/) - API documentation

### AI Governance:
- [AI Ethics Guidelines](https://www.oecd.ai/)  
- [Responsible AI Development](https://www.microsoft.com/ai/responsible-ai)

---

## 🎓 Завершение трека AI Agent Development

Поздравляем! Вы завершили трек **AI Agent Track** для KMP разработки.

### Что вы освоили:
✅ Настройка AI-инструментов и prompt engineering  
✅ Генерация KMP кода (expect/actual, Compose UI, models)
✅ Проектирование архитектуры с AI  
✅ AI-ассистированное тестирование (90%+ coverage)
✅ Нативные интеграции (Camera, Location, ML Kit, Core ML)
✅ Production-ready AI workflows и security

### Ваши следующие шаги:

1. **Практика:** Используйте AI ежедневно в реальной разработке
2. **Углубление:** Изучайте advanced topics (LLM fine-tuning, custom models)
3. **Обучение других:** Делитесь знаниями с командой через workshops  
4. **Вклад в сообщество:** Делитесь prompt'ами и best practices

### Рекомендуемые следующие треки:
- [Senior Track](../Senior_Track/README.md) - Углубленное изучение KMP
- [Architecture Track](../Architecture_Track/README.md) - Advanced architecture patterns

---

## 🏆 Сертификат завершения

После завершения всех 6 модулей и практических заданий вы можете получить сертификат:

**Сертификат AI Agent Development для Kotlin Multiplatform**

Вы успешно завершили трек и продемонстрировали компетенции в:
- AI-assisted KMP development
- Prompt engineering для mobile разработки  
- Security-conscious AI workflows
- Production-ready AI integration

**Уровень:** Advanced  
**Часов обучения:** 50+ часов
**Дата выдачи:** [DATE]

---

## 🤝 Вклад в сообщество

Делитесь своим опытом:
1. **GitHub:** Отправьте PR с вашими prompt'ами в `prompt_library/`
2. **Blog:** Напишите статью о вашем опыте использования AI в KMP
3. **Conference:** Предложите talk на KotlinConf или local meetup

### Что можно поделиться:
- Эффективные prompt'ы для специфичных задач
- Сравнение разных AI инструментов  
- Case studies ускорения разработки с AI
- Best practices и anti-patterns

---

## 📞 Поддержка и сообщество

- **Discord:** [KMP Learn Community](https://discord.gg/kmp-learn)
- **GitHub Issues:** [Report problems или suggest improvements](https://github.com/KMP-Learn/AI_Agent_Track/issues)
- **Email:** support@kmp-learn.dev

---

**Удачи в вашей AI-powered KMP разработке! 🚀🤖✨**

*Помните: AI - это мощный инструмент, но вы остаетесь архитектором и ответственным инженером. Используйте его мудр!*

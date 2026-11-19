# 📝 Самопроверка: Модуль 6 — AI в Production

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 11+ из 15 без подглядывания.

---

## Вопрос 1. Какой приоритет шаблон code review отводит безопасности?

- **A)** Security Issues — критично; Performance — высокий; Code Quality и Testing — средний; Kotlin Best Practices — низкий
- **B)** Все категории равнозначны
- **C)** Сначала Kotlin Best Practices, потом безопасность
- **D)** Приоритет только у Performance

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаблон из темы 6.1 задаёт роль «Senior Principal Engineer с 10+ лет опытом, эксперт в security и code quality» и пять критериев в порядке убывания приоритета: 1) Security Issues (критично), 2) Performance Issues (высокий), 3) Code Quality (средний), 4) Testing Coverage (средний), 5) Kotlin Best Practices (низкий).

**Почему другие не подходят:**
- B — шаблон явно ранжирует категории
- C — Kotlin Best Practices стоит последней
- D — Performance на втором месте после безопасности

</details>

---

## Вопрос 2. [code] Какую уязвимость AI нашёл в этом методе и что предложил?

```kotlin
suspend fun updateUser(userId: String, data: Map<String, Any>) {
    val query = "UPDATE users SET ${data.keys.joinToString(",") { "$it" }} = ? WHERE id = '$userId'"
    db.execute(query, *data.values.toTypedArray())
}
```

- **A)** SQL Injection; исправление — параметризованные запросы через SQLDelight (`db.users.updateUser(userId, data)`)
- **B)** Race condition; исправление — `upsert`
- **C)** Cache stampede; исправление — `mutex`
- **D)** Уязвимостей нет

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В ожидаемом результате AI ставит это на первое место: `WHERE id = '$userId'` — интерполяция в SQL, атакующий может передать `admin' OR '1'='1`. Рекомендованный fix — параметризованные запросы через SQLDelight: `db.users.updateUser(userId, data)` (SQLDelight сам экранирует параметры).

**Почему другие не подходят:**
- B — race condition найден в другом методе — `getUser()` (get → insert)
- C — cache stampede — отдельный High Priority пункт про параллельные cache miss
- D — это ровно та уязвимость, из-за которой Security score равен 3/10

</details>

---

## Вопрос 3. [code] Какие проблемы AI выявил в `getUser()`?

```kotlin
val cached = db.users.get(userId)
if (cached != null) return cached
val response = apiService.getUser(userId)
if (response.success) {
    val user = response.data
    db.users.insert(user) // Potential race condition
    return user
}
```

- **A)** Race condition между `get()` и `insert()` → `upsert`/транзакция; плюс cache stampede → mutex или distributed lock
- **B)** Только SQL injection в `insert(user)`
- **C)** Только несогласованность имен
- **D)** Только блокировку главного потока

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

AI указывает: «Race Condition в Cache Update» — между `get()` и `insert()` другая корутина может вставить ту же запись, fix — `db.transaction { ... db.users.upsert(user) }`. Вторым пунктом идёт «Cache Stampede» — при cache miss все запросы одновременно идут в API, fix — защита через mutex или распределённый лок.

**Почему другие не подходят:**
- B — SQL injection относится к `updateUser()`
- C — inconsistent naming — отдельная рекомендация 🟢 уровня
- D — blocking main thread — категория Performance, здесь не упоминается

</details>

---

## Вопрос 4. Каков формат вывода шаблона code review?

- **A)** 🔴 Critical Issues (Security) → 🟡 High Priority (Performance) → 🟢 Recommendations (Best Practices) → Summary с числом issues и Security score /10
- **B)** Один общий список замечаний без приоритетов
- **C)** Только таблица с покрытием тестами
- **D)** Оценка по 100-балльной шкале без разбора кода

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаблон требует строго секционированный отчёт: Critical (Security), High Priority (Performance), Recommendations (Best Practices) и Summary с «Total issues found: X critical, Y high, Z medium», «Security score: /10» и Recommended actions. Каждое issue — с файлом, строкой, описанием проблемы и конкретным кодом исправления.

**Почему другие не подходят:**
- B — без приоритетов теряется главное — критичные угрозы
- C — тесты — лишь один из пяти критериев
- D — шкала /10, а не /100, и код исправлений обязателен

</details>

---

## Вопрос 5. Вопрос-ловушка. Какой Security score получил `UserRepository` в разборе примера?

- **A)** 3/10 — с пометкой «Fix SQL injection перед deployment!»
- **B)** 8/10
- **C)** 10/10
- **D)** 0/10

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Ловушка на путаницу с критерием задания 6.3. В **разборе примера** AI поставил 3/10 из-за критичных уязвимостей и написал: «Action Required: Fix SQL injection перед deployment!». А 8/10 — это порог, которого нужно достичь в практическом задании 6.3 (Security Audit).

**Почему другие не подходят:**
- B — целевой порог задания, а не результат примера
- C — в примере 2 critical issue, такая оценка невозможна
- D — в отчёте указано именно 3/10

</details>

---

## Вопрос 6. [prompt] Из чего состоит «хороший prompt» по стандартам Team AI Guidelines?

- **A)** Роль, Контекст, Задача, Ограничения, Формат
- **B)** Достаточно одной фразы «Напиши repository для User»
- **C)** Только роль и задача
- **D)** Только контекст и формат вывода

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Раздел «Prompt Engineering Standards» перечисляет пять элементов: 1) Роль («Ты — Senior Kotlin разработчик...»), 2) Контекст проекта и архитектуры, 3) чёткая задача, 4) ограничения (tech stack, версии), 5) формат вывода. Пример хорошего prompt'а содержит все пять: роль KMP-разработчика, CRUD через `Result`, SQLDelight, in-memory cache с 5 мин TTL, Flow, Kotlin 1.9+, Clean Architecture, KDoc.

**Почему другие не подходят:**
- B — именно такой короткий prompt даёт непредсказуемый результат
- C, D — неполный набор

</details>

---

## Вопрос 7. Что входит в шаблон API documentation (тема 6.2)?

- **A)** Overview, Installation, Quick Start, API Reference (Signature, Description, Parameters, Returns, Throws, Example), Best Practices, Common Pitfalls, Related
- **B)** Badge, Screenshots, Key features, Contributing, License
- **C)** Только KDoc-комментарии в коде
- **D)** Incident Response и список одобренных инструментов

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаблон API documentation задаёт именно такую структуру с таблицей параметров (Name | Type | Required | Description), секциями Returns, Throws, Example с error handling, Best Practices, Common Pitfalls и Related. Требуется документация для всех public классов и функций.

**Почему другие не подходят:**
- B — это структура README.md
- C — KDoc — результат/инструмент, а не шаблон документа
- D — разделы Team AI Guidelines

</details>

---

## Вопрос 8. Вопрос-ловушка. Где в шаблонах документации встречается «Architecture overview с diagram»?

- **A)** В структуре `README.md` (пункт 7), а не в шаблоне API documentation
- **B)** В разделе API Reference
- **C)** В Security Checklist перед deployment
- **D)** В отчёте security audit

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Ловушка на смешивание двух шаблонов темы 6.2. «Architecture overview с diagram» — пункт 7 структуры README (вместе с badges, features, requirements, installation, quick start, module structure, contributing, license). В шаблоне API документации такого раздела нет.

**Почему другие не подходят:**
- B — API Reference содержит Signature, Parameters, Returns, Throws, Example
- C, D — это документы по безопасности и процессам, а не документация API

</details>

---

## Вопрос 9. Что относится к платформенно-специфичной безопасности в шаблоне security audit?

- **A)** Android: backup prevention, `flagSecure`; iOS: Keychain, sandboxing
- **B)** Только TLS certificate pinning
- **C)** Только обфускация R8/ProGuard
- **D)** Только rate limiting

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаблон audit'а из темы 6.3 содержит пять аспектов: Data Security, Authentication & Authorization, Network Security, Code Security и Platform-Specific Security. Последний включает Android (backup prevention, flagSecure) и iOS (Keychain usage, sandboxing). Для каждого аспекта нужно проверить реализацию, выявить риски и предложить исправления с кодом.

**Почему другие не подходят:**
- B — это Data/Network Security (encryption in transit)
- C — раздел Code Security
- D — раздел Network Security

</details>

---

## Вопрос 10. Что согласно AI Usage Guidelines требует обязательного REVIEW REQUIRED?

- **A)** Security-critical код (auth, encryption, payments), бизнес-логика с финансовыми последствиями, data access/persistence layer, сетевой код
- **B)** Boilerplate, data classes и DTO
- **C)** Генерация secrets и credentials
- **D)** Объяснение сложных концепций

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Гайдлайн делит всё на три группы: ✅ DO (boilerplate, тесты, документация, объяснения, рефакторинг), ⚠️ REVIEW REQUIRED (security-critical код, финансовые расчеты, слой данных, сетевой код), ❌ DON'T (deploy без human review, security-реализации без audit, 100% доверие AI, генерация secrets/credentials).

**Почему другие не подходят:**
- B — это категория DO
- C — прямо запрещено (DON'T)
- D — категория DO

</details>

---

## Вопрос 11. [config] Что из перечисленного **НЕ** входит в Security Checklist перед deployment?

- **A)** «Code review approval: 100%»
- **B)** API keys и secrets в environment variables
- **C)** TLS certificate pinning configured
- **D)** Code obfuscation enabled для production builds

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Ловушка на смежные чек-листы. Security Checklist перед deployment содержит шесть пунктов: secrets в env variables, TLS certificate pinning, шифрование БД, отсутствие debug-кода и логирования чувствительных данных, обфускация для production, завершённый security audit с исправленными critical issue. А «Code review approval: 100%» — это метрика качества из Team AI Guidelines (Quality Metrics).

**Почему другие не подходят:**
- B, C, D — все три дословно есть в Security Checklist

</details>

---

## Вопрос 12. Какой процесс ревью установлен для Level 2 — Medium Risk?

- **A)** Self-review + peer review (2 человека) + security checklist
- **B)** Self-review + peer review (1 человек)
- **C)** Self-review + 2 peers + security audit + architecture review
- **D)** Только self-review

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Уровни из Team AI Guidelines: Level 1 (Low Risk: boilerplate, тесты, документация, простые UI) — self-review + 1 peer; Level 2 (Medium Risk: бизнес-логика, use cases, repository, network layer) — self-review + 2 peers + security checklist; Level 3 (High Risk: security, платежи, data access) — self-review + 2 peers + security audit + architecture review.

**Почему другие не подходят:**
- B — это Level 1
- C — это Level 3
- D — self-review один не достаточно ни для одного уровня

</details>

---

## Вопрос 13. Какие метрики качества заданы для AI-generated code?

- **A)** Test coverage ≥80%, Code review approval 100%, Security audit Passed для Level 2+, Performance benchmarks meet SLA
- **B)** Test coverage 90%+, остальное по усмотрению
- **C)** Coverage 100% и обязательный audit для всех уровней
- **D)** Метрик нет — всё решает человек вручную

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Раздел Quality Metrics задаёт четыре порога: покрытие тестами ≥80%, одобрение code review 100%, Security audit Passed для кода Level 2+ и performance-бенчмарки в рамках SLA.

**Почему другие не подходят:**
- B — 90%+ встречается в описании освоенного трека (AI-ассистированное тестирование), но не как порог Quality Metrics
- C — audit требуется для Level 2+, а не для Level 1
- D — метрики прямо зафиксированы в гайдлайне

</details>

---

## Вопрос 14. Какой инструмент **НЕ** одобрен для production-кода?

- **A)** Бесплатный (free tier) AI-инструмент
- **B)** GitHub Copilot (enterprise license)
- **C)** Cursor.sh (team license)
- **D)** Claude 3.5 Sonnet для сложных задач

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В разделе Approved AI Tools: Primary — GitHub Copilot (enterprise license) и Cursor.sh (team license); Secondary — Claude 3.5 Sonnet (complex tasks) и Continue.dev (local models); Not Approved — free tier AI tools для production code и unknown/unvetted AI services.

**Почему другие не подходят:**
- B, C — primary tools
- D — secondary tool

</details>

---

## Вопрос 15. AI сгенерировал проблемный код. Каков верный порядок действий по Incident Response?

- **A)** Не deploy → Report в `#ai-incidents` Slack → Analyze root cause (bad prompt? галлюцинация?) → Update guidelines
- **B)** Сначала задеплоить, чтобы проверить в проде, затем откатить
- **C)** Удалить файл и забыть
- **D)** Переписать всё вручную без разбора причин

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Раздел Incident Response задаёт ровно четыре шага: 1) Не deploy в production; 2) сообщить в Slack-канал `#ai-incidents`; 3) проанализировать первопричину (плохой prompt или галлюцинация AI); 4) обновить гайдлайны, чтобы ошибка не повторилась. Это согласуется с общим правилом: AI — ассистент, а не замена инженеру, и deploy без human review запрещён.

**Почему другие не подходят:**
- B — деплой проблемного кода прямо запрещён первым пунктом
- C — без анализа root cause ошибка повторится
- D — ручной переписывающий не отменяет анализ причин и обновление гайдлайнов

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 11–15 | 🎉 Отлично! Ты готов применять AI в production безопасно — поздравляем с завершением трека |
| 8–10 | 📖 Перечитай темы 6.3 и 6.4: guardrails, уровни ревью и метрики качества |
| 5–7 | ⚠️ Пройди модуль ещё раз. Сфокусируйся на security audit и правилах AI Usage Guidelines |
| < 5 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🏆 Модуль 6 завершает **AI Agent Track**. После всех 6 модулей и практических заданий — сертификат AI Agent Development для Kotlin Multiplatform (уровень Advanced, 50+ часов).

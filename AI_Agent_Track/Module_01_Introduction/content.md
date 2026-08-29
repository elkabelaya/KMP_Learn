# Модуль 1: Введение в AI-агентов для разработки

## 📋 Обзор модуля

**Продолжительность:** 2 недели  
**Сложность:** Beginner  
**Цель:** Настроить окружение и освоить основы работы с AI-инструментами

---

## 🎯 Цели обучения

После прохождения модуля вы сможете:
- ✅ Настроить GitHub Copilot, Cursor и другие AI-инструменты
- ✅ Писать эффективные промпты для генерации кода
- ✅ Понимать возможности и ограничения AI-агентов

---

## 📚 Темы модуля

### Тема 1.1: Обзор инструментов (3 часа)

#### GitHub Copilot
**Что это:** AI-ассистент от Microsoft, интегрированный в IDE

**Возможности:**
- Автодополнение кода на основе контекста
- Генерация целых функций по комментариям
- Предложения по refactoring

**Установка:**
```bash
# В Android Studio / IntelliJ IDEA:
1. Settings → Plugins → Marketplace
2. Поиск "GitHub Copilot"
3. Install и авторизация через GitHub
```

**Пример использования:**
```kotlin
// Напишите комментарий:
// Функция для валидации email адреса

// Copilot предложит:
fun validateEmail(email: String): Boolean {
    return email.matches(Regex("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"))
}
```

#### Cursor
**Что это:** AI-first редактор кода (форк VS Code)

**Возможности:**
- Чат с файлами проекта в контексте
- Генерация кода по описанию задачи
- Multi-file editing с AI

**Установка:**
```bash
# Скачать с cursor.sh
1. Установить Cursor
2. Настроить API key (OpenAI, Anthropic, или бесплатный)
3. Открыть KMP проект
4. Нажать Cmd+L (Ctrl+L) для открытия чата
```

**Пример промпта:**
```
@commonMain @androidMain Создай expect/actual для работы с файловой системой. 
Ожидаемый интерфейс: fun readFile(path: String): String?
```

#### Claude / ChatGPT
**Что это:** Чат-интерфейсы для сложных задач

**Когда использовать:**
- Проектирование архитектуры
- Объяснение сложных концепций
- Code review и поиск багов

**Пример промпта:**
```
Я разрабатываю KMP приложение с Clean Architecture. 
У меня есть 3 слоя: Domain, Data, UI.

Объясни, как правильно организовать dependency injection 
с помощью Koin для commonMain и платформенных модулей.

Дай пример кода с:
- Repository интерфейс в Domain
- Repository реализация в Data  
- Koin модули для каждого слоя
```

---

### Тема 1.2: Настройка окружения (4 часа)

#### Шаг 1: GitHub Copilot в Android Studio

```bash
# 1. Установите плагин
Settings → Plugins → Marketplace → GitHub Copilot

# 2. Авторизация
Tools → GitHub Copilot → Sign in with GitHub

# 3. Настройка
Settings → Tools → GitHub Copilot:
- ✓ Enable code completion
- ✓ Enable inline suggestions
- ✓ Enable chat (если доступно)
```

#### Шаг 2: Cursor Editor

```bash
# 1. Скачайте с cursor.sh

# 2. Настройте API (в настройках Cursor):
- LLM Provider: Anthropic Claude 3.5 Sonnet (рекомендуется)
- Или OpenAI GPT-4o
- Или бесплатный tier

# 3. Создайте .cursorrules в корне проекта:
```

**Пример `.cursorrules`:**
```markdown
# Kotlin Multiplatform Project Rules

## Code Style
- Use Kotlin conventions (camelCase, PascalCase для классов)
- Prefer data classes for DTOs
- Use sealed interfaces for result types

## Architecture
- Follow Clean Architecture (Domain, Data, UI)
- Use Repository pattern for data access
- Dependency Injection with Koin

## KMP Specific
- Common code in commonMain
- Platform-specific in androidMain/iosMain
- Use expect/actual for platform differences

## Testing
- Write unit tests for business logic
- Use Coroutines TestDispatcher in tests
```

#### Шаг 3: Claude API Key (опционально)

```bash
# 1. Зарегистрируйтесь на console.anthropic.com
# 2. Создайте API key
# 3. Сохраните в .env файл (не коммитьте в Git!)

# В ~/.cursorrules добавьте:
CLAUDE_API_KEY=your-api-key-here
```

---

### Тема 1.3: Основы промпт-инжиниринга (5 часов)

#### Структура эффективного промпта

**Формула:** Контекст + Задача + Ограничения + Формат вывода

**Плохой промпт:**
```
Напиши функцию для валидации формы
```

**Хороший промпт:**
```
Контекст: Я разрабатываю KMP приложение для Android и iOS с Compose UI.

Задача: Создай функцию валидации формы регистрации пользователя.

Ограничения:
- Email должен быть валидным (содержать @ и домен)
- Пароль минимум 8 символов, содержать букву и цифру
- Имя не пустое, минимум 2 символа
- Вернуть Result type с четкой типизацией ошибок

Формат вывода: Kotlin код с KDoc комментариями
```

#### Few-shot prompting

**Пример:**
```kotlin
// Пример 1: Валидация email
fun validateEmail(email: String): ValidationResult {
    return if (email.contains("@")) Valid else Invalid("Email должен содержать @")
}

// Пример 2: Валидация пароля  
fun validatePassword(password: String): ValidationResult {
    return if (password.length >= 8) Valid else Invalid("Пароль минимум 8 символов")
}

// Теперь создай функцию валидации имени пользователя:
```

#### Chain-of-thought prompting

**Пример:**
```
Задача: Спроектируй архитектуру для кроссплатформенного приложения.

Думай пошагово:
1. Какие слои нужны в Clean Architecture?
2. Что должно быть в commonMain?
3. Какие expect/actual нужны для платформенных особенностей?
4. Как организовать DI с Koin?

Теперь напиши структуру проекта...
```

#### Итеративное уточнение

**Диалог с AI:**
```
Вы: Создай Repository для пользователей
AI: [генерирует код]

Вы: Хорошо, но добавь кэширование в памяти с LRU cache
AI: [обновляет код]

Вы: Теперь добавь обработку ошибок сети с retry policy
AI: [обновляет код]

Вы: Отлично! Теперь напиши unit-тесты для этого Repository
AI: [генерирует тесты]
```

---

## 📝 Практические задания

### Задание 1.1: Настройка окружения (2 часа)

**Цель:** Установить и настроить все AI-инструменты

**Шаги:**
1. Установите GitHub Copilot в Android Studio
2. Скачайте и настройте Cursor
3. Создайте `.cursorrules` файл в проекте KMP_Learn
4. Протестируйте автодополнение кода

**Критерии успеха:**
- ✓ Copilot предлагает код при вводе комментариев
- ✓ Cursor открывается и работает с KMP проектом
- ✓ `.cursorrules` создан с правилами для вашего проекта

---

### Задание 1.2: Генерация KMP проекта (3 часа)

**Цель:** Создать базовый KMP проект с помощью ИИ

**Промпт для Cursor/Claude:**
```
Создай структуру KMP проекта с:
- commonMain: Domain слой (interfaces, models)
- androidMain: Android приложение с Compose
- iosMain: iOS приложение с Compose

Включи:
1. Базовую структуру Gradle (build.gradle.kts)
2. Ожидаемый интерфейс для Platform API
3. Пример expect/actual для получения имени платформы

Используй Kotlin 1.9+, Compose Multiplatform 1.5+
```

**Критерии успеха:**
- ✓ Проект компилируется для Android и iOS
- ✓ expect/actual работает корректно
- ✓ Код следует Kotlin conventions

---

### Задание 1.3: Практика промпт-инжиниринга (4 часа)

**Цель:** Научиться писать эффективные промпты

**Упражнения:**

1. **Генерация data class:**
```
Создай data class User для KMP приложения с полями:
- id (UUID)
- email (String, валидация)
- name (String, не пустое)
- createdAt (Instant)

Добавь companion object с factory function для создания тестовых данных.
```

2. **Генерация функции:**
```
Напиши функцию для сортировки списка привычек по:
- Активность (active/inactive)
- Дата создания (newest first)

Используй Kotlin sortWith и Comparator.
Добавь KDoc с примерами использования.
```

3. **Генерация тестов:**
```
Напиши unit-тесты для функции validateEmail из Задания 1.2:
- Тест на валидный email
- Тест на невалидный email (без @)
- Тест на пустую строку

Используй JUnit 5 и Coroutines TestDispatcher.
```

**Критерии успеха:**
- ✓ Все 3 промпта дают рабочий код с первого раза
- ✓ Код компилируется и проходит тесты
- ✓ Вы понимаете, почему промпт сработал

---

## 📊 Проверка знаний

### Quiz (10 вопросов)

1. **Что такое few-shot prompting?**
   - a) Один пример в промпте
   - b) Несколько примеров для обучения паттерну ✓
   - c) Генерация нескольких вариантов кода

2. **Какой инструмент лучше для multi-file editing?**
   - a) GitHub Copilot
   - b) Cursor ✓
   - c) ChatGPT

3. **Что должно быть в `.cursorrules`?**
   - a) API keys
   - b) Правила код-стайла и архитектуры ✓
   - c) Логирование запросов

4. **Какая формула эффективного промпта?**
   - a) Задача + Код
   - b) Контекст + Задача + Ограничения + Формат ✓
   - c) Только описание задачи

5. **Когда НЕ стоит использовать ИИ?**
   - a) Для boilerplate кода
   - b) Для критической бизнес-логики без проверки ✓
   - c) Для генерации тестов

(и ещё 5 вопросов...)

---

## 🎯 Итоговый проект модуля

**Задача:** Создать мини-проект "AI Code Generator" с помощью ИИ

**Требования:**
1. KMP проект с 3 слоями (Domain, Data, UI)
2. Минимум 50% кода сгенерировано ИИ
3. Все промпты задокументированы в `PROMPTS.md`
4. Code review сгенерированного кода

**Критерии оценки:**
- ✓ Проект компилируется и запускается
- ✓ Код следует best practices
- ✓ Промпты эффективные и повторяемые

---

## 📚 Дополнительные материалы

### Статьи:
- [GitHub Copilot Documentation](https://docs.github.com/en/copilot)
- [Cursor AI Guide](https://docs.cursor.sh/)
- [Prompt Engineering Guide](https://www.promptingguide.ai/)

### Видео:
- "Kotlin Multiplatform with AI" (YouTube, 45 мин)
- "Cursor Editor Tutorial for KMP" (YouTube, 30 мин)

### Практика:
- [Exercism Kotlin Track](https://exercism.org/tracks/kotlin) — решайте задачи с AI-помощью
- [LeetCode](https://leetcode.com/) — алгоритмы с AI code review

---

## 🚀 Следующий шаг

Переходите к [Модулю 2](../Module_02_AI_UI/content.md): AI-помощник в разработке UI

**Время до следующего модуля:** 2 недели  
**Рекомендуемая практика:** Ежедневно использовать AI-инструменты хотя бы 30 минут

---

**Удачи в освоении AI-агентов! 🤖✨**

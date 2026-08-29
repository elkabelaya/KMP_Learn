# Модуль 1: Введение в AI-агентную разработку

## 📋 Обзор модуля

**Продолжительность:** 4 часа  
**Сложность:** Beginner  
**Цель:** Настройка AI-среды и освоение основ prompt engineering для разработки

---

## 🎯 Цели обучения

После прохождения модуля вы сможете:
- ✅ Настроить **AI-инструменты** (Copilot, Cursor, Claude)
- ✅ Писать **эффективные prompt'ы** для генерации кода
- ✅ Понимать **ограничения и риски** AI-генерации
- ✅ Создать свой первый KMP проект с помощью AI

---

## 📚 Темы модуля

### Тема 1.1: Обзор AI-инструментов для разработки (30 минут)

#### Популярные инструменты:

**1. GitHub Copilot ($10/мес)**
- Inline code completion в VS Code / IntelliJ
- Chat interface для вопросов и генерации кода
- Интеграция с GitHub

**2. Cursor.sh (Free / $20/мес)**
- AI-powered IDE на базе VS Code
- Advanced chat с контекстом всего проекта
- File editing через natural language

**3. Claude 3.5 Sonnet (Free tier / $20/мес)**
- Лучший для сложных задач и архитектуры
- Большое контекстное окно (200K tokens)
- Отличное понимание кода

**4. Continue.dev (Free)**
- VS Code extension с поддержкой разных моделей
- Local models через Ollama
- Customizable workflows

---

### Тема 1.2: Основы Prompt Engineering (1 час)

#### Структура эффективного prompt'а:

```
[Роль] + [Контекст] + [Задача] + [Ограничения] + [Формат вывода]
```

**Пример плохого prompt'а:**
```
Создай KMP проект
```

**Пример хорошего prompt'а:**
```
Ты - Senior Kotlin Multiplatform разработчик с 5+ лет опытом.

Контекст:
- Создаю приложение для трекинга эко-привычек (EcoTrack)
- Целевые платформы: Android, iOS, Desktop
- Использую Compose Multiplatform для UI

Задача:
Создай структуру KMP проекта с модулями:
- :app (Android)
- :iosApp (iOS)  
- :shared (KMP common code)

Ограничения:
- Kotlin 1.9.20+
- Gradle 8.4+
- Clean Architecture (domain, data, ui слои)

Формат вывода:
1. Дерево структуры проекта
2. build.gradle.kts для каждого модуля
3. Краткое описание назначения каждого модуля
```

---

#### Техники prompt engineering:

**1. Few-shot prompting (примеры):**
```
Создай expect/actual для работы с файловой системой.

Пример 1 - SharedPreferences:
expect interface Preferences {
    fun getString(key: String): String?
    fun putString(key: String, value: String)
}

actual class Preferences actual constructor(...) { ... }

Пример 2 - UserDefaults:
expect interface Storage {
    fun save(data: String, key: String)
    fun load(key: String): String?
}

Теперь создай для FileIO с методами read() и write().
```

**2. Chain of Thought (пошаговое мышление):**
```
Создай repository для работы с привычками.

Подумай пошагово:
1. Какие данные нужно хранить? (id, title, category, streak)
2. Какие операции нужны? (CRUD + поиск по категории)
3. Как кэшировать данные? (in-memory cache с TTL)
4. Как обрабатывать ошибки? (sealed class Result)

Теперь напиши код.
```

**3. Role prompting (роль эксперта):**
```
Ты - Principal Engineer в JetBrains, специализируешься на KMP.
Объясни как правильно структурировать большой KMP проект с 10+ платформами.
```

---

### Тема 1.3: Настройка AI-среды (45 минут)

#### Установка GitHub Copilot в VS Code:

1. Установите расширение "GitHub Copilot"
2. Авторизуйтесь через GitHub
3. Настройте подписку ($10/мес или free для студентов)

#### Установка Cursor.sh:

```bash
# Скачать с cursor.sh
# Установить как обычную IDE
# Настроить API key (Anthropic Claude или OpenAI)
```

#### Настройка Continue.dev:

В `.continue/config/continue.config.js`:
```javascript
module.exports = {
  models: [
    {
      title: "Claude 3.5 Sonnet",
      apiBase: "https://api.anthropic.com",
      apiKey: process.env.ANTHROPIC_API_KEY,
    },
  ],
  tabAutocompleteModel: "Claude 3.5 Sonnet",
};
```

---

### Тема 1.4: Создание первого KMP проекта с AI (1.5 часа)

#### Практическое задание: Создайте проект "Hello KMP" с AI

**Step 1: Генерация структуры проекта**

Промпт для AI:
```
Создай минимальный KMP проект с Compose Multiplatform.

Требования:
- commonMain с простым "Hello World" Composable
- androidMain с Application class и Activity
- iosApp с SwiftUI wrapper

Создай:
1. settings.gradle.kts с plugin management
2. build.gradle.kts для root и shared модуля
3. commonMain/com/example/greeting/Greeting.kt с Composable
4. androidMain composableApp {}
5. iosApp.swift с ComposeView

Покажи полный код каждого файла.
```

**Step 2: Запуск на Android и iOS**

Промпт для AI:
```
Как запустить этот KMP проект на Android эмуляторе и iOS симуляторе?
Дай пошаговые команды для:
1. Android Studio
2. Xcode
3. Командной строки (gradle)

Объясни возможные ошибки и как их исправить.
```

**Step 3: Добавление фичи с AI**

Промпт для AI:
```
Добавь в Greeting.kt кнопку, которая меняет текст при нажатии.

Используй:
- StateHoisting с remember { mutableStateOf(...) }
- Material3 Button компонент
- ClickListener

Покажи полный обновленный код Greeting.kt.
```

---

## 📝 Практические задания модуля

### Задание 1.1: Настройка окружения (30 минут)

**Требования:**
- Установите минимум 2 AI-инструмента (Copilot + Cursor или Claude)
- Протестируйте каждый на простой задаче

**Критерии:**
- ✓ Copilot предлагает автодополнение кода
- ✓ Cursor/Claude отвечает на вопросы о коде

---

### Задание 1.2: Создание библиотеки prompt'ов (45 минут)

**Требования:**
Создайте файл `prompt_library.md` с 5+ эффективными prompt'ами:

```markdown
# Моя библиотека AI Prompt'ов для KMP

## Генерация expect/actual
```
[ваш prompt]
```

Результат: [описание что получилось]

## Создание Composable компонентов
```
[ваш prompt]
```

Результат: [описание что получилось]
```

**Критерии:**
- ✓ 5+ prompt'ов разных категорий
- ✓ Каждый prompt протестирован и работает

---

### Задание 1.3: Создание "Hello KMP" проекта (1 час)

**Требования:**
- Используйте AI для генерации 80%+ кода
- Самостоятельно настройте и запустите проект

**Критерии:**
- ✓ Проект компилируется на Android и iOS
- ✓ Показывает "Hello World" с кнопкой

---

## 🚫 Ошибки новичков при работе с AI

### ❌ Копируете код без понимания
**Решение:** Всегда читайте и понимайте сгенерированный код перед использованием

### ❌ Слишком vague prompt'ы
**Решение:** Добавляйте контекст, ограничения и примеры

### ❌ Доверяете AI на 100%
**Решение:** Всегда тестируйте сгенерированный код

### ❌ Не итерируете prompt'ы
**Решение:** Улучшайте prompt'ы на основе результатов

---

## 📚 Дополнительные материалы

### Статьи:
- [Prompt Engineering Guide](https://www.promptingguide.ai/)
- [GitHub Copilot Documentation](https://docs.github.com/en/copilot)

### Видео:
- "AI Pair Programming in 2024" (YouTube, 35 мин)
- "Building KMP Apps with AI" (KotlinConf, 40 мин)

### Инструменты:
- [PromptPerfect](https://promptperfect.jina.ai/) - Улучшение prompt'ов
- [FlowGPT](https://flowgpt.com/) - Библиотека prompt'ов

---

## 🚀 Следующий шаг

Переходите к [Модулю 2](../Module_02_AI_Code_Generation/content.md): Генерация KMP кода с AI

**Время до следующего модуля:** 1 неделя  
**Рекомендуемая практика:** Ежедневно используйте AI для хотя бы одной задачи

---

## 💡 Pro Tips

### Совет 1: Контекст - это всё
Чем больше контекста вы даёте AI, тем лучше результат:
- Архитектура проекта
- Существующие паттерны в коде
- Ограничения и требования

### Совет 2: Итерируйте prompt'ы
Редко получается идеально с первого раза. Уточняйте:
```
Хорошо, но добавь обработку ошибок и логирование.
Теперь сделай код более идиоматичным для Kotlin.
Добавь KDoc комментарии.
```

### Совет 3: Создавайте шаблоны prompt'ов
Сохраняйте успешные prompt'ы для повторного использования:

```markdown
## Шаблон: Создание Repository

Ты - Senior KMP разработчик. Создай repository для [ENTITY] с:
- CRUD операциями через sealed class Result
- In-memory кэшированием с TTL [TIME]
- SQLDelight для persistence
- Flow для реактивных обновлений

Следуй Clean Architecture паттерну.
```

---

**Удачи в освоении AI-агентной разработки! 🤖✨**

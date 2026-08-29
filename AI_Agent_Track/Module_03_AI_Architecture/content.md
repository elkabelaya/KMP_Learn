# Модуль 3: AI для архитектуры и дизайна

## 📋 Обзор модуля

**Продолжительность:** 10 часов  
**Сложность:** Intermediate  
**Цель:** Использование AI для проектирования архитектуры, рефакторинга и создания ADR

---

## 🎯 Цели обучения

После прохождения модуля вы сможете:
- ✅ Проектировать **Clean Architecture** с AI-ассистентом
- ✅ Создавать **Architecture Decision Records (ADR)** автоматически
- ✅ Рефакторить код с помощью AI
- ✅ Оптимизировать **структуру проекта** и разделение модулей

---

## 📚 Темы модуля

### Тема 3.1: Проектирование архитектуры с AI (2 часа)

#### Шаблон prompt'а для проектирования:

```
Ты - Principal Architect с 10+ лет опытом в мобильной разработке.

Контекст:
- Приложение: [НАЗВАНИЕ] - [ОПИСАНИЕ]
- Платформы: Android, iOS, Desktop (KMP)
- Ожидаемая нагрузка: [КОЛИЧЕСТВО] пользователей
- Критические требования: [СПИСОК]

Задача:
Спроектируй архитектуру приложения с использованием Clean Architecture.

Требования:
1. Модульная структура (feature-first или layer-first?)
2. Dependency Injection стратегия
3. Data flow (Unidirectional, StateFlow, etc.)
4. Offline-first стратегия
5. Синхронизация данных

Выведи:
1. Дерево структуры проекта с объяснением каждого модуля
2. Diagram в Mermaid формате для визуализации архитектуры
3. Список технологий и библиотек с обоснованием выбора
4. Потенциальные риски и как их mitigровать

Обоснуй каждый архитектурный выбор.
```

---

#### Пример: Проектирование архитектуры для E-commerce приложения

**Промпт:**
```
Спроектируй архитектуру для KMP e-commerce приложения.

Требования:
- 100K+ пользователей
- Offline-first с background sync
- Real-time уведомления о заказах
- Payment integration (Stripe, PayPal)
- Analytics и A/B testing

Создай модульную структуру с feature modules.
```

**Ожидаемый результат от AI:**

```
# Архитектура E-commerce KMP приложения

## Структура проекта

ecommerce-kmp/
├── :app (Android entry point)
├── :iosApp (iOS entry point)  
├── :desktopApp (Desktop entry point)
├── :shared/
│   ├── :core/
│   │   ├── :core-common (Result, Either, extensions)
│   │   ├── :core-network (Ktor client, interceptors)
│   │   ├── :core-db (SQLDelight setup, migrations)
│   │   └── :core-designsystem (Material 3 theme, components)
│   ├── :features/
│   │   ├── :feature-auth (Login, registration, OAuth)
│   │   ├── :feature-products (Catalog, search, filters)
│   │   ├── :feature-cart (Shopping cart management)
│   │   ├── :feature-checkout (Payment, order placement)
│   │   └── :feature-profile (User settings, orders history)
│   └── :domain/
│       ├── :domain-common (UseCases interface, errors)
│       └── :domain-models (Entities, value objects)
└── build.gradle.kts

## Архитектурные решения

### 1. Feature-First Modularization
Каждая фича - отдельный модуль с полной вертикальной срезкой:
- feature-products/
  - domain/ (UseCases, entities)
  - data/ (Repository implementations, local/remote)
  - ui/ (Compose screens, viewModels)

Преимущества:
- Independent compilation и testing
- Team scalability (разные команды над разными фичами)
- Clear ownership и boundaries

### 2. Unidirectional Data Flow
UI -> Event -> ViewModel -> State -> UI

Преимущества:
- Predictable state management
- Easy testing и debugging
- Consistent pattern across features

### 3. Offline-First с Conflict Resolution
- Local DB как source of truth
- Background sync с сервером
- Last-write-wins + manual conflict resolution для critical data

### 4. Technology Stack
- DI: Koin (lightweight, good KMP support)
- DB: SQLDelight (type-safe queries, multiplatform)
- Network: Ktor Client (unified HTTP API)
- State: StateFlow + MVI pattern

## Риски и митигация

| Риск | Вероятность | Impact | Митигация |
|------|-------------|--------|-----------|
| Module coupling | Medium | High | Strict dependency rules, architecture tests |
| Build time degradation | High | Medium | Gradle configuration cache, build metrics monitoring |
| iOS performance | Medium | High | Native optimizations через expect/actual |
```

---

### Тема 3.2: Создание ADR с AI (2 часа)

#### Шаблон Architecture Decision Record:

```
Ты - Technical Lead. Создай ADR (Architecture Decision Record) для [РЕШЕНИЕ].

Формат ADR:
# [ADR-001] [Название решения]

## Статус
[Proposed | Accepted | Deprecated | Superseded]

## Контекст
[Описание ситуации, проблемы или возможности]

## Решение
[Конкретное решение которое принято]

## Последствия
### Положительные
- [Преимущества решения]

### Отрицательные  
- [Недостатки и компромиссы]

## Альтернативы
### [Альтернатива 1]
[Почему не выбрали]

### [Альтернатива 2]
[Почему не выбрали]

## Ссылки
- [Related documents, RFCs, discussions]

Дата: 2025-01-XX
Автор: [Имя]
```

---

#### Пример ADR для выбора DI фреймворка:

**Промпт:**
```
Создай ADR для выбора Dependency Injection фреймворка в KMP проекте.

Контекст:
- Кроссплатформенное приложение (Android, iOS, Desktop)
- Требуется DI в commonMain и платформенных модулях
- Команда: 5 разработчиков, средний опыт

Рассмотри варианты: Koin, Kodein, Injecting
Выбери лучший вариант и обоснуй.

Создай полный ADR в стандартном формате.
```

**Ожидаемый результат:** AI создаст полный ADR с обоснованием выбора Koin.

---

### Тема 3.3: Рефакторинг кода с AI (2 часа)

#### Шаблон prompt'а для рефакторинга:

```
Ты - Senior Kotlin разработчик, эксперт в clean code и рефакторинге.

Задача:
Рефактори следующий код следуя принципам SOLID и Kotlin best practices.

[ВСТАВИТЬ КОД]

Требования к рефакторингу:
1. Уменьши cyclomatic complexity
2. Раздели большие функции на меньшие с понятными именами
3. Устрани code duplication через extraction
4. Улучши naming (descriptive names)
5. Добавь KDoc комментарии где нужно
6. Улучши error handling

Для каждого изменения объясни:
- Что было не так в оригинальном коде
- Как рефакторинг улучшает код
- Какие принципы clean code применяем

Покажи:
1. Рефакторированный код
2. Список примененных refactoring patterns
3. Сравнение метрик (строк кода, complexity) до и после
```

---

#### Пример рефакторинга:

**До (плохой код):**
```kotlin
// Monolithic function - 150+ строк
suspend fun processOrder(orderData: String): Boolean {
    // Parsing, validation, DB operations, API calls, notifications все в одной функции
    // 150+ строк кода...
}
```

**Промпт для AI:**
```
Рефактори эту монолитную функцию следуя Single Responsibility Principle.

Раздели на:
1. ParseOrderInput - парсинг и валидация
2. CreateOrderUseCase - бизнес-логика создания заказа
3. SaveOrderLocally - persistence
4. SyncOrderToServer - network operation  
5. SendNotifications - user notifications

Каждая функция должна быть <= 30 строк с четкой ответственностью.
```

**После (рефакторировано):** AI создаст 5 отдельных функций с четкими границами.

---

### Тема 3.4: Оптимизация структуры проекта (2 часа)

#### Шаблон prompt'а для оптимизации:

```
Ты - Staff Engineer, эксперт в KMP архитектуре.

Контекст:
- Текущая структура проекта: [ОПИСАНИЕ]
- Проблемы: [Медленная сборка, tight coupling, etc.]
- Цели: [Ускорить build, улучшить maintainability]

Задача:
Предложи оптимизацию структуры проекта.

Анализируй:
1. Module dependencies (циклические зависимости?)
2. Build time bottlenecks
3. Code organization (feature-first vs layer-first)
4. Shared code percentage

Предложи:
1. Новую структуру проекта с обоснованием изменений
2. Миграционный план (step-by-step)
3. Ожидаемые улучшения (build time %, maintainability score)

Создай Mermaid diagram для визуализации до/после.
```

---

## 📝 Практические задания модуля

### Задание 3.1: Спроектировать архитектуру нового фича (2 часа)

**Требования:**
- Используйте AI для проектирования архитектуры "Social Sharing" фичи
- Создайте ADR с обоснованием решений

**Критерии:**
- ✓ Четкая модульная структура
- ✓ Обоснование каждого решения

---

### Задание 3.2: Рефакторинг существующего кода (3 часа)

**Требования:**
- Выберите 2-3 больших функции из EcoTrack (>50 строк)
- Рефакторите с помощью AI

**Критерии:**
- ✓ Уменьшение cyclomatic complexity на 30%+
- ✓ Улучшение читаемости

---

### Задание 3.3: Оптимизация структуры проекта (3 часа)

**Требования:**
- Проанализируйте текущую структуру с AI
- Предложите и примените оптимизации

**Критерии:**
- ✓ Ускорение build time на 20%+
- ✓ Улучшение модульности

---

### Задание 3.4: Создание библиотеки ADR (2 часа)

**Требования:**
Создайте 3-5 ADR для вашего проекта:

1. ADR-001: Выбор DI фреймворка
2. ADR-002: Стратегия error handling  
3. ADR-003: Offline-first синхронизация
4. ADR-004: Модульная структура

**Критерии:**
- ✓ Все ADR в стандартном формате
- ✓ Обоснование каждого решения

---

## 🚫 Ошибки при использовании AI для архитектуры

### ❌ Слепое доверие AI рекомендациям
**Решение:** Всегда проверяйте архитектурные решения на соответствие вашим требованиям

### ❌ Игнорирование trade-offs
**Решение:** Просите AI явно описывать компромиссы каждого решения

### ❌ Отсутствие контекста
**Решение:** Давайте полный контекст: команда, сроки, бюджет, constraints

---

## 📚 Дополнительные материалы

### Книги:
- "Architecture Patterns with Kotlin" (Dmitry Jemelin)
- "Fundamentals of Software Architecture" (Mark Richards, Neal Ford)

### Шаблон ADR:
- [Michael Nygard's ADR template](https://adr.github.io/)

### Инструменты:
- [Mermaid Live Editor](https://mermaid.live/) - Diagram generation
- [ArchUnit](https://www.archunit.org/) - Architecture tests

---

## 🚀 Следующий шаг

Переходите к [Модулю 4](../Module_04_AI_Testing/content.md): AI-ассистированное тестирование

**Время до следующего модуля:** 1-2 недели  
**Рекомендуемая практика:** Создавайте минимум 1 ADR в неделю

---

**Удачи в проектировании архитектуры с помощью ИИ! 🏗️🤖**

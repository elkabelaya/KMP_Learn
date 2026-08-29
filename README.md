# 🎓 Kotlin Multiplatform + Compose Multiplatform: Полный курс

**От нуля до Middle-разработчика за 8 модулей через создание реального приложения**

---

## 📱 О проекте: EcoTrack

**EcoTrack** — кроссплатформенное приложение для отслеживания эко-привычек.

### Макеты:
https://www.figma.com/design/ftUrIiC2aFrWhINEU5g1j0/EcoTracker

### Функционал:
- ✅ Добавление и управление привычками (CRUD)
- ✅ Категории: транспорт, мусор, энергия, вода, еда
- ✅ Трекер прогресса и статистика
- ✅ Система достижений (бейджи)
- ✅ Локальная БД с синхронизацией
- ✅ Push-уведомления и напоминания
- ✅ Deep links для открытия конкретных экранов
- ✅ Экспорт/импорт данных
- ✅ Темная тема и адаптивность

### Технологии:
- **Kotlin Multiplatform (KMP)** — общая бизнес-логика
- **Compose Multiplatform** — UI для Android, iOS и Desktop
- **SQLDelight** — типобезопасная БД
- **Ktor Client** — HTTP-запросы
- **Koin** — Dependency Injection
- **Coroutines + Flow** — асинхронность

---

## 📚 Структура курса

### [Модуль 1: Введение и настройка окружения](./Module_01_Introduction/content.md)
**Что изучим:**
- Установка Kotlin, Android Studio, Xcode
- Создание KMP проекта с Gradle
- Понимание структуры проекта (commonMain, androidMain, iosMain)

**Практика:** Создание "Hello World" приложения для Android и iOS

---

### [Модуль 2: Основы Compose Multiplatform](./Module_02_Compose_Basics/content.md)
**Что изучим:**
- Declarative UI и Composable функции
- Layouts: Column, Row, Box, LazyColumn
- Material 3 компоненты (Card, Button, TextField)
- State hoisting и @Composable

**Практика:** Экран списка привычек с базовым UI

---

### [Модуль 3: Архитектура и Dependency Injection](./Module_03_Architecture/content.md)
**Что изучим:**
- Clean Architecture (Domain, Data, UI слои)
- Repository Pattern
- Koin для DI в commonMain и платформенных модулях

**Практика:** Рефакторинг приложения по Clean Architecture

---

### [Модуль 4: Работа с данными (SQLDelight)](./Module_04_Data_Layer/content.md)
**Что изучим:**
- SQLDelight и типобезопасные запросы
- Миграции БД
- Expect/Actual для платформенных драйверов (SQLite, SQLCipher)

**Практика:** Локальное хранение привычек в БД

---

### [Модуль 5: Сетевое взаимодействие (Ktor Client)](./Module_05_Networking/content.md)
**Что изучим:**
- Ktor Client для HTTP-запросов
- JSON сериализация (kotlinx.serialization)
- Интерцепторы (Auth, Logging, Retry)
- Offline-first архитектура

**Практика:** Синхронизация данных с сервером (с mock API)

---

### [Модуль 6: Продвинутый UI и Визуализация](./Module_06_Advanced_UI/content.md)
**Что изучим:**
- Кастомные компоненты на Canvas API
- Анимации (AnimatedVisibility, updateTransition)
- Дизайн-система в commonMain

**Практика:** Графики статистики и анимированные бейджи

---

### [Модуль 7: Нативные фичи и Интеграции](./Module_07_Native_Features/content.md)
**Что изучим:**
- Expect/Actual для нативных API
- Push-уведомления (FCM для Android, APNs для iOS)
- Deep links
- Работа с файловой системой

**Практика:** Уведомления о привычках и экспорт/импорт данных

---

### [Модуль 8: Тестирование, Оптимизация и Релиз](./Module_08_Testing_and_Release/content.md)
**Что изучим:**
- Unit-тесты для commonMain (JUnit 5, Kotest)
- UI-тесты с Compose Test
- Оптимизация размера APK и времени запуска
- Публикация в Google Play и App Store

**Практика:** Написание тестов и подготовка к релизу

---

## 🚀 Как проходить курс

### Рекомендуемый порядок:
1. **Прочитайте теорию** модуля (30-60 минут)
2. **Изучите примеры кода** в content.md
3. **Выполните практическое задание** (2-4 часа)
4. **Переходите к следующему модулю** только после сдачи текущего

### Время прохождения:
- **Модуль 1:** ~3 часа
- **Модуль 2:** ~6 часов
- **Модуль 3:** ~8 часов
- **Модуль 4:** ~10 часов
- **Модуль 5:** ~12 часов
- **Модуль 6:** ~10 часов
- **Модуль 7:** ~15 часов
- **Модуль 8:** ~20 часов

**Итого:** ~84 часа (около 3-4 недель при изучении по 5-7 часов в день)

---

## 📋 Требования к системе

### Для Android разработки:
- **OS:** Windows 10/11, macOS 12+, Linux
- **RAM:** 16GB+ (рекомендуется)
- **Android Studio:** Hedgehog (2023.1.1) или новее
- **JDK:** 17

### Для iOS разработки:
- **OS:** macOS 13+ (Ventura или новее)
- **Xcode:** 15.0+
- **RAM:** 16GB+ (рекомендуется)

### Для KMP разработки:
- **Kotlin:** 1.9.20+
- **Gradle:** 8.4+

---

## 🎯 Что вы получите после курса

### Hard Skills:
- ✅ Уверенное владение Kotlin Multiplatform
- ✅ Опыт разработки кроссплатформенных приложений (Android + iOS)
- ✅ Понимание Clean Architecture и паттернов проектирования
- ✅ Навыки работы с БД, сетью, нативными API
- ✅ Умение писать unit-тесты и UI-тесты

### Soft Skills:
- ✅ Умение разбивать сложные задачи на модули
- ✅ Навык чтения документации и поиска решений
- ✅ Понимание процесса релиза приложения

### Портфолио:
- ✅ Готовое приложение **EcoTrack** для Android и iOS
- ✅ Исходный код на GitHub с полной историей коммитов
- ✅ Возможность опубликовать в Google Play и App Store

---

## 📖 Дополнительные ресурсы

### Официальная документация:
- [Kotlin Multiplatform](https://kotlinlang.org/docs/multiplatform.html)
- [Compose Multiplatform](https://www.jetbrains.com/help/compose-multiplatform/)
- [SQLDelight](https://cashapp.github.io/sqldelight/)
- [Ktor Client](https://ktor.io/docs/getting-started-ktor-client.html)

### Книги:
- "Kotlin Multiplatform by Example" (O'Reilly)
- "Android Programming: The Big Nerd Ranch Guide"

### YouTube каналы:
- [KotlinConf](https://www.youtube.com/c/KotlinConf)
- [JetBrains Compose](https://www.youtube.com/@jetbrainscompose)

### Сообщества:
- [Kotlin Slack](https://kotlinlang.slack.com/) (каналы #multiplatform, #compose)
- [Kotlin Forum](https://discuss.kotlinlang.org/)

---

## 🤝 Вклад в проект

Если вы нашли ошибку или хотите улучшить курс:
1. Создайте Issue с описанием проблемы
2. Отправьте Pull Request с исправлением

### Что можно улучшить:
- Добавить больше примеров кода
- Улучшить объяснения сложных тем
- Добавить видео-уроки к модулям
- Перевести на другие языки

---

## 📝 Лицензия

Этот курс распространяется под лицензией **MIT**. Вы можете свободно использовать, изменять и распространять материалы.

**Автор:** Ваш AI-ассистент  
**Дата создания:** 2024  
**Версия курса:** 1.0

---

## 📚 Полный обзор курса

Хотите увидеть полную картину обучения от Junior до Senior?
- [📖 COURSE SUMMARY](./COURSE_SUMMARY.md) - Complete overview обоих треков (Main Course + Senior Track)
- [📋 IMPLEMENTATION SUMMARY](./IMPLEMENTATION_SUMMARY.md) - Детальный обзор всей структуры курса и созданных материалов

---

## 🎉 Начните обучение прямо сейчас!

Переходите к [Модулю 1](./Module_01_Introduction/content.md) и создайте свое первое KMP приложение!

---

## 🚀 После завершения курса: Продвинутые треки

После прохождения основного курса (8 модулей), у вас есть **два пути** для дальнейшего роста:

### Option 1: [🎓 Senior KMP Developer Track](./Senior_Track/README.md)
**10 продвинутых модулей для перехода от Middle к Senior:**

- **Module 01-03:** Advanced KMP, Compose Multiplatform, Database patterns
- **Module 04-05:** Native integration, Performance optimization  
- **Module 06-07:** Security best practices, Monorepo architecture
- **Module 08-09:** Advanced testing, CI/CD & DevOps
- **Module 10:** Mentorship и professional growth

**Время прохождения:** ~250 часов (3-4 месяца)  
**Результат:** Готовность к позиции Senior KMP Developer

---

### Option 2: [🤖 AI Agent Track](./AI_Agent_Track/README.md) ⭐ NEW!
**Комплексный курс по разработке с AI агентами (Cursor, Copilot):**

- **Module 01:** Foundation & AI tool setup
- **Module 02:** Code generation prompts для KMP  
- **Module 03:** Testing & security с AI assistance
- **Module 04:** Documentation & production readiness
- **Module 05:** Advanced prompt engineering patterns

**Бонусные материалы:**
- 📚 16-week learning path от junior до senior с AI
- 🔒 Comprehensive security checklist для production apps  
- 🚀 Quick reference guide для daily use
- 📊 Real-world examples и case studies

**Время прохождения:** ~16 недель (гибкий график)  
**Результат:** Увеличьте продуктивность в 3-5x с AI assistance

---

### Какой трек выбрать?

| Ваш профиль | Рекомендуемый путь |
|------------|-------------------|
| Хотите глубокое понимание KMP internals | **Senior Track** |
| Хотите ускорить разработку с AI tools | **AI Agent Track** |
| Есть время и на оба (идеально!) | **Оба трека последовательно** |
| Уже senior, хотите master AI | **AI Agent Track** (Module 5) |

> 💡 **Совет:** Многие разработчики проходят оба трека - сначала Senior Track для fundamentals, затем AI Agent Track для productivity boost!

---

**Удачи в изучении Kotlin Multiplatform! 🚀**

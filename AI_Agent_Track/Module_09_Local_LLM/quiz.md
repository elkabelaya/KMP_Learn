# 📝 Самопроверка: Модуль 9 — Локальные LLM модели

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 13+ из 15 без подглядывания.

---

## Вопрос 1. В чём главное преимущество локальных моделей согласно модулю?

- **A)** Качество выше, чем у GPT-4 / Claude 3.5
- **B)** Данные не покидают устройство — конфиденциальность
- **C)** Вообще не требуют RAM и GPU
- **D)** Работают только при наличии интернета

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Тема 9.1 перечисляет преимущества: 🔒 конфиденциальность (данные не покидают устройство), 🚫 нет лимитов API, 📴 оффлайн-работа, ⚡ низкая задержка, 🎛️ полный контроль (кастомизация и fine-tuning).

**Почему другие не подходят:**
- A — это как раз ограничение: локальные модели меньше по параметрам и уступают GPT-4/Claude 3.5 по качеству
- C — ограничение прямо обратное: требуется RAM 8–24 ГБ, а для больших моделей ещё и GPU
- D — наоборот, одна из причин использовать локальные модели — работа без интернета

</details>

---

## Вопрос 2. Каково типичное контекстное окно локальных моделей (как ограничение в модуле)?

- **A)** 8K–32K токенов
- **B)** 100K+ токенов
- **C)** 4K токенов
- **D)** Не ограничено

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В разделе «Ограничения» темы 9.1 прямо сказано: контекстное окно обычно **8K–32K токенов** против 100K+ в cloud. Из-за этого в теме 9.4 и появляются Sliding Window, Summary Compression и RAG.

**Почему другие не подходят:**
- B — это характеристики cloud-моделей
- C — 4K указано только у Phi-3 Mini в таблице сравнения (4K–128K), но это не типичное значение
- D — ограничение контекста — одна из ключевых проблем локальных моделей

</details>

---

## Вопрос 3. Какой формат квантования GGUF модуль рекомендует для баланса качество/размер?

- **A)** Q4_K_M
- **B)** Q5_K_M
- **C)** Q8_0
- **D)** FP16

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Тема 9.1, форматы GGUF:

| Формат | Характеристика |
|--------|----------------|
| **Q4_K_M** | **рекомендуется** — 4-bit, баланс качество/размер |
| Q5_K_M | 5-bit, лучшее качество, больше размер |
| Q8_0 | 8-bit, почти без потерь, большой размер |

FP16 вообще не является квантованием — это «без квантования».

**Почему другие не подходят:**
- B — лучше качество, но больше размер, рекомендуемым не назван
- C — почти без потерь, но тяжёлый
- D — FP16 это исходная точность, а не сжатие (у Llama 3.1 8B это 16 ГБ)

</details>

---

## Вопрос 4. Размеры Llama 3.1 8B из модуля:

```
Q4_K_M:  4.7 GB
Q5_K_M:  5.4 GB
Q8_0:    6.9 GB
FP16:   16.0 GB
```

Сколько весит модель в рекомендуемом формате и во сколько раз FP16 больше него?

- **A)** 4.7 ГБ; FP16 примерно в 3,4 раза больше
- **B)** 6.9 ГБ; FP16 ровно в 2 раза больше
- **C)** 5.4 ГБ; FP16 ровно в 3 раза больше
- **D)** 16 ГБ; Q4_K_M в 4 раза меньше

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Рекомендуемый формат — Q4_K_M, то есть **4.7 ГБ**. Отношение: `16 / 4.7 ≈ 3.4`. Именно поэтому задание 9.3 предлагает скачать Llama 3.1 8B в Q4_K_M (~4.7 ГБ) для ноутбука — модель укладывается в память и не вызывает OOM.

**Почему другие не подходят:**
- B — 6.9 ГБ это Q8_0, а не рекомендуемый формат
- C — 5.4 ГБ это Q5_K_M; к тому же 16 / 5.4 ≈ 3, а не ровно 3
- D — 16 ГБ это FP16; 16 / 4.7 ≈ 3.4, а не 4

</details>

---

## Вопрос 5. Какой инструмент модуль называет «самым простым способом» запустить локальную модель?

- **A)** llama.cpp
- **B)** Ollama
- **C)** Text Generation WebUI
- **D)** LM Studio

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Заголовок темы 9.2: **«Ollama — самый простой способ»**. Установка — одна команда `curl -fsSL https://ollama.com/install.sh | sh`, дальше `ollama pull llama3.1` и `ollama run llama3.1`, выход из интерактива — `/bye`.

**Почему другие не подходят:**
- A — llama.cpp это «максимальная производительность», требует сборки из исходников с флагами
- C — Text Generation WebUI это «полноценный сервер» с клонированием репозитория и `pip install`
- D — LM Studio это GUI с API-сервером, удобный, но в модуле «самым простым» назван Ollama

</details>

---

## Вопрос 6. Какой адрес и эндпоинт использует REST API Ollama в примере модуля?

```bash
ollama serve
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.1",
  "prompt": "Напиши Kotlin data class User"
}'
```

- **A)** `http://localhost:8000/api/generate`
- **B)** `http://localhost:11434/api/generate`
- **C)** `http://localhost:8080/api/chat`
- **D)** `http://localhost:11434/v1/completions`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`ollama serve` автоматически поднимает REST API на **localhost:11434**. В модуле показаны `/api/generate` (с полями `model`, `prompt`, в ответе `response` и `done`) и `/api/chat` (с `messages`). Тот же порт используется в androidMain-реализации `LocalLLMServiceImpl`.

**Почему другие не подходят:**
- A — 8000 это порт оркестратора из модуля 8 (FastAPI), ловушка на смежный модуль
- C — неверный порт; `/api/chat` существует, но на 11434
- D — такого эндпоинта в модуле нет

</details>

---

## Вопрос 7. Что означают флаги сборки llama.cpp из примера?

```bash
make LLAMA_CUBLAS=1   # ...
make LLAMA_METAL=1    # ...
make LLAMA_VULKAN=1   # ...
```

- **A)** CUBLAS — NVIDIA CUDA, METAL — Apple Metal (macOS), VULKAN — Linux/Windows
- **B)** CUBLAS — уровень квантования, METAL — GUI-режим, VULKAN — Docker-образ
- **C)** Все три — разные версии формата GGUF
- **D)** CUBLAS — только CPU, METAL — только iOS, VULKAN — только веб

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Тема 9.4 (GPU Acceleration):

- `make LLAMA_CUBLAS=1` — NVIDIA CUDA
- Apple Metal на macOS поддерживается автоматически, флаг `make LLAMA_METAL=1` показан при сборке из исходников
- `make LLAMA_VULKAN=1` — Linux/Windows

После сборки GPU включается запуском с `--n-gpu-layers 99`.

**Почему другие не подходят:**
- B — флаги RELATED к GPU, а не к квантованию/GUI/Docker
- C — GGUF это формат файла модели, он задаётся при конвертации (`convert-hf-to-gguf.py`) и квантовании (`./quantize ... Q4_K_M`)
- D — CUBLAS наоборот включает GPU (CUDA), а не CPU-only режим

</details>

---

## Вопрос 8. Какие методы объявлены в интерфейсе `LocalLLMService` в commonMain?

```kotlin
interface LocalLLMService {
    suspend fun generate(prompt: String): LLMResponse
    suspend fun chat(messages: List<ChatMessage>): ChatResponse
    fun isAvailable(): Boolean
    fun getModelInfo(): ModelInfo?
}
```

- **A)** `generate`, `chat`, `isAvailable`, `getModelInfo`
- **B)** `generate`, `chat`, `compress`, `cache`
- **C)** `generate`, `refine`, `isAvailable`, `getModelInfo`
- **D)** `generate`, `chat`, `isAvailable`, `loadModel`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Это контракт, который реализуется в `androidMain`, `iosMain` и `desktopMain` (подход expect/actual из целей модуля). Рядом определены данные:

- `LLMResponse(content, model, tokensUsed, generationTime)`
- `ChatMessage(role: MessageRole, content)` и `enum class MessageRole { USER, ASSISTANT, SYSTEM }`
- `ChatResponse(message, model, tokensUsed)`
- `ModelInfo(name, size, parameters, quantization)`

**Почему другие не подходят:**
- B — `compress` и `cache` это отдельные менеджеры/обёртки из темы 9.4, не методы интерфейса
- C — `refine` есть у `CloudLLMService` в гибридном подходе
- D — `loadModel` в интерфейсе не объявлен

</details>

---

## Вопрос 9. Как устроена androidMain-реализация `LocalLLMServiceImpl`?

```kotlin
private val client = HttpClient(CIO)

val response = client.post("http://localhost:11434/api/generate") {
    contentType(ContentType.Application.Json)
    setBody("""{"model": "llama3.1", "prompt": "$prompt", "stream": false}""")
}.body<GenerateResponse>()
```

- **A)** Ktor Client с engine CIO, POST на `/api/generate` с JSON-телом; ошибки оборачиваются в `LLMException`
- **B)** URLSession с нативным Network framework
- **C)** Прямой вызов llama.cpp через `ProcessBuilder`
- **D)** OkHttp + ручной парсинг без data class-ов

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Код из темы 9.3: `HttpClient(CIO)`, POST с `contentType(ContentType.Application.Json)`, `setBody` с полями `model`, `prompt`, `stream: false`, десериализация в `GenerateResponse(model, response, done)`. При ошибке бросается `LLMException("Failed to generate: ${e.message}")`. `isAvailable()` проверяет порт через `Socket("localhost", 11434)`, а `getModelInfo()` пока возвращает `null` с `TODO`.

**Почему другие не подходят:**
- B — это iosMain-реализация (URLSession / Network framework)
- C — это desktopMain-подход
- D — OkHttp в модуле не используется, ответы разбираются в типизированные `GenerateResponse` / `ChatApiResponse`

</details>

---

## Вопрос 10. Какой подход используется в desktopMain-реализации для генерации?

```kotlin
val process = ProcessBuilder(
    "./llama/main",
    "-m", "models/llama-3.1-8b.Q4_K_M.gguf",
    "-p", prompt,
    "-n", "512"
).start()
val output = process.inputStream.bufferedReader().readText()
```

- **A)** Прямой запуск llama.cpp как subprocess (альтернатива — HTTP-запрос к Ollama)
- **B)** Нативный `URLSession`
- **C)** Ktor Client с CIO engine
- **D)** GraphQL-запрос к облачному API

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В теме 9.3 прямо указано: Desktop может использовать прямой вызов llama.cpp через JNI или HTTP-клиент как Android; в примере — запуск `./llama/main` через `ProcessBuilder` с моделью, промптом и `-n 512`. Результат оборачивается в `LLMResponse` с подсчётом `estimateTokens` и времени генерации.

**Почему другие не подходят:**
- B — iOS-подход
- C — Android-подход
- D — про облако в этой реализации речи нет; модуль про локальные модели

</details>

---

## Вопрос 11. Что делает `SlidingWindowContextManager` при превышении лимита токенов?

- **A)** Очищает всю историю и начинает диалог заново
- **B)** Удаляет самую старую пару сообщений user–assistant, пока сумма токенов не уложится в `maxTokens = 8000`
- **C)** Сжимает старые сообщения в краткое резюме через LLM
- **D)** Переключает запрос на cloud-провайдер

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Код из темы 9.4:

```kotlin
val currentTokens = messages.sumOf { estimateTokens(it.content) }
if (currentTokens > maxTokens) trimToMaxTokens()

// внутри trimToMaxTokens():
val indexToRemove = findOldestUserAssistantPair()
messages.removeAt(indexToRemove)
messages.removeAt(indexToRemove + 1)
```

Удаляется именно **пара** (старое сообщение пользователя и ответ ассистента), пока не станет ≤ `maxTokens` (по умолчанию 8000).

**Почему другие не подходят:**
- A — история не очищается полностью, удаляются только самые старые пары
- C — это другой метод — `SummaryContextManager`
- D — переключение на cloud относится к гибридному подходу из темы 9.5

</details>

---

## Вопрос 12. ⚠️ Что делает `SummaryContextManager` с историей сообщений?

```kotlin
val oldMessages = messages.take(messages.size - 4)
val recentMessages = messages.takeLast(4)

val summaryPrompt = """
    Сожми следующие сообщения в краткое резюме (максимум 200 символов):
    ${oldMessages.joinToString("\n") { "${it.role}: ${it.content}" }}
"""
val summary = llm.generate(summaryPrompt)

return listOf(
    ChatMessage(MessageRole.SYSTEM, summary.content),
) + recentMessages
```

- **A)** Резюме добавляется с ролью `USER`, сохраняются последние 4 сообщения
- **B)** Резюме добавляется с ролью `SYSTEM`, сохраняются последние 4 сообщения
- **C)** Резюме добавляется с ролью `ASSISTANT`, сохраняются последние 2 сообщения
- **D)** Резюме заменяет всю историю, последние сообщения не сохраняются

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Это вопрос-ловушка. Легко решить, что сжатая история «становится сообщением пользователя», но в коде явно указано `ChatMessage(MessageRole.SYSTEM, summary.content)`. Последние 4 сообщения (`takeLast(4)`) сохраняются как есть, а старые (`take(size - 4)`) сжимаются в резюме не более 200 символов.

**Почему другие не подходят:**
- A — неверная роль: `USER` здесь не используется (ловушка)
- C — неверны и роль, и число сохраняемых сообщений
- D — recentMessages всё равно добавятся в результат

</details>

---

## Вопрос 13. Как `RAGContextManager` выбирает данные для контекста?

- **A)** `vectorStore.search(query, topK = 3)`, затем промпт с блоками «Контекст» и «Вопрос»
- **B)** `vectorStore.search(query, topK = 10)`, затем ответ без промпта
- **C)** Берёт последние 4 сообщения чата и сжимает их
- **D)** Удаляет самую старую пару сообщений, пока не останется 8000 токенов

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Тема 9.4, третий подход к контексту — RAG (Retrieval Augmented Generation):

```kotlin
val relevantDocs = vectorStore.search(query, topK = 3)
val contextPrompt = """
    Используя следующую информацию, ответь на вопрос:
    Контекст:
    ${relevantDocs.joinToString("\n\n")}
    Вопрос: $query
    Ответ:
"""
return llm.generate(contextPrompt)
```

**Почему другие не подходят:**
- B — topK равен 3, а не 10; без промпта обойтись нельзя
- C — это SummaryContextManager
- D — это SlidingWindowContextManager

</details>

---

## Вопрос 14. Что вернёт `CachedLLMService` для промптов «Привет» и «привет»?

```kotlin
val cacheKey = normalizePrompt(prompt) // lowercase().replace(Regex("\\s+"), " ").trim()
return cache.getOrPut(cacheKey) {
    delegate.generate(prompt).also { saveToPersistentCache(cacheKey, response) }
}
```

- **A)** Два разных запроса к `delegate.generate`
- **B)** Один и тот же закэшированный ответ (второй раз модель не вызывается)
- **C)** Ошибка: ключи кэша должны быть уникальными
- **D)** Кэш сработает, только если промпты совпадают символ в символ

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

`normalizePrompt` приводит текст к нижнему регистру, схлопывает любые последовательности пробелов в один и обрезает края. Значит «Привет» и «привет» дают одинаковый ключ, и `getOrPut` вернёт уже сохранённый `LLMResponse`. Дополнительно ответ асинхронно пишется в persistent cache.

**Почему другие не подходят:**
- A — именно этого кэш и избегает (экономия времени при повторных запросах)
- C — коллизия тут желаемая, а не ошибка
- D — ловушка: нормализация как раз снимает требование точного совпадения

</details>

---

## Вопрос 15. ⚠️ Промпт: «Проанализируй транзакцию 1234567812345678». Какой провайдер выберет `HybridLLMService.selectProvider`?

```kotlin
if (containsSensitiveData(prompt)) return LLMProvider.LOCAL
if (isComplexTask(prompt)) return LLMProvider.CLOUD
if (isRoutineTask(prompt)) return LLMProvider.LOCAL
if (isQualityCritical(prompt)) return LLMProvider.HYBRID
return LLMProvider.LOCAL
```

- **A)** `CLOUD`, потому что слово «анализируй» делает задачу сложной
- **B)** `LOCAL`, потому что 16-значный номер — конфиденциальные данные, и эта проверка идёт первой
- **C)** `HYBRID`, потому что требуется высокое качество
- **D)** `CLOUD`, потому что рутинные задачи уходят в облако

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Это вопрос-ловушка на **порядок правил**. Промпт содержит и «анализируй» (`isComplexTask` → CLOUD), и 16 цифр подряд (`containsSensitiveData` проверяет `Regex("\\b\\d{16}\\b")` → LOCAL). Но проверка конфиденциальных данных стоит **первой** и сразу делает `return`, поэтому до `isComplexTask` дело не доходит.

Напоминание остальных правил: рутинные задачи → LOCAL, критичное качество → HYBRID (локальный черновик + `cloud.refine`), по умолчанию → LOCAL. В HYBRID-ветке локальная модель делает черновик, а облачная его улучшает.

**Почему другие не подходят:**
- A — ловушка: сложность действительно обнаружена, но проверка конфиденциальности имеет приоритет
- C — `isQualityCritical` не сработал раньше, а HYBRID — не вариант по умолчанию
- D — рутинные задачи, наоборот, уходят на LOCAL; CLOUD здесь не выбран ни по одному правилу

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 13–15 | 🎉 Отлично! Квантование, инструменты и гибридный подход — твои. Ты готов к итоговому проекту трека |
| 10–12 | 📖 Хорошо. Перечитай темы 9.3 (expect/actual) и 9.4 (оптимизация), посмотри вопросы, где ошибся |
| 7–9 | ⚠️ Пройди модуль ещё раз. Сфокусируйся на форматах GGUF, инструментах запуска и стратегиях контекста |
| < 7 | 🆘 Давай разберём вместе — перечитай модуль и вернись с конкретными вопросами |

---

> 🏁 Это финальный квиз AI Agent Development Track. Итоговый проект: KMP-приложение с multi-agent системой и локальной LLM. Удачи! 🔒🤖

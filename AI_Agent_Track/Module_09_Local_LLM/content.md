# Модуль 9: Локальные LLM модели

## 📋 Обзор модуля

**Продолжительность:** 3 недели  
**Сложность:** Advanced  
**Цель:** Освоить запуск и интеграцию локальных LLM моделей в KMP приложения для конфиденциальной работы без облачных API

---

## 🎯 Цели обучения

После прохождения модуля вы сможете:
- ✅ Запускать **локальные LLM модели** (Llama 3, Mistral, Phi-3) на своем устройстве
- ✅ Интегрировать локальные модели в **KMP приложения** через expect/actual
- ✅ Оптимизировать производительность (квантование, GPU acceleration)
- ✅ Реализовать **гибридный подход** (локальная + cloud модели)
- ✅ Обеспечить **конфиденциальность данных** без отправки в облако

---

## 📚 Темы модуля

### Тема 9.1: Обзор локальных моделей (4 часа)

#### Почему локальные модели?

**Преимущества:**
- 🔒 **Конфиденциальность** — данные не покидают устройство
- 🚫 **Нет лимитов API** — unlimited запросы без оплаты
- 📴 **Оффлайн работа** — не зависит от интернета
- ⚡ **Низкая задержка** — нет network overhead
- 🎛️ **Полный контроль** — кастомизация и fine-tuning

**Ограничения:**
- ⚠️ **Меньше параметров** — ниже качество чем GPT-4/Claude 3.5
- ⚠️ **Требует ресурсов** — RAM (8-24GB), GPU для больших моделей
- ⚠️ **Контекст окно** — обычно 8K-32K токенов vs 100K+ в cloud

#### Сравнение популярных моделей:

| Модель | Параметры | Размер (Q4) | Контекст | Сильные стороны |
|--------|-----------|-------------|----------|-----------------|
| **Llama 3.1 8B** | 8B | 4.7GB | 8K-128K | Баланс качество/размер, код |
| **Mistral 7B** | 7B | 4.1GB | 8K-32K | Отличный для кода, быстрый |
| **Phi-3 Mini** | 3.8B | 2.3GB | 4K-128K | Компактная, быстрая на CPU |
| **Qwen 2.5 7B** | 7B | 4.1GB | 32K | Multilingual, код |
| **Gemma 2 9B** | 9B | 5.4GB | 8K | Google, good reasoning |
| **Llama 3.1 70B** | 70B | 40GB | 128K | Максимальное качество, требует GPU |

#### Квантование (Quantization)

**Что это:** Сжатие модели с минимальной потерей качества

**Форматы GGUF:**
- **Q4_K_M** (рекомендуется) — 4-bit, баланс качество/размер
- **Q5_K_M** — 5-bit, лучшее качество, больше размер
- **Q8_0** — 8-bit, почти без потерь, большой размер

**Пример размеров Llama 3.1 8B:**
- Q4_K_M: 4.7GB
- Q5_K_M: 5.4GB
- Q8_0: 6.9GB
- FP16 (без квантования): 16GB

---

### Тема 9.2: Запуск локальных моделей (6 часов)

#### Ollama — самый простой способ

**Установка:**
```bash
# macOS / Linux
curl -fsSL https://ollama.com/install.sh | sh

# Windows
# Скачать с ollama.com и установить .exe
```

**Запуск моделей:**
```bash
# Скачать и запустить Llama 3.1
ollama pull llama3.1
ollama run llama3.1

# В интерактивном режиме:
>>> Напиши Kotlin функцию для валидации email
func validateEmail(email: String): Boolean {
    return email.matches(Regex("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"))
}

# Выйти: /bye
```

**Запуск как API сервер:**
```bash
# Ollama автоматически запускает REST API на localhost:11434
ollama serve

# Запрос через curl:
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.1",
  "prompt": "Напиши Kotlin data class User"
}'

# Ответ:
{
  "model": "llama3.1",
  "response": "data class User(\n  val id: UUID,\n  val email: String\n)",
  "done": true
}
```

**Популярные модели в Ollama:**
```bash
ollama pull llama3.1        # 8B, баланс
ollama pull mistral         # 7B, код
ollama pull phi3            # 3.8B, компактная
ollama pull qwen2.5         # 7B, multilingual
ollama pull llama3.1:70b    # 70B, максимальное качество (требуется GPU)
```

---

#### LM Studio — GUI с API сервером

**Установка:**
1. Скачать с lmstudio.ai
2. Установить и запустить

**Использование:**
1. **Search & Download**: Поиск моделей в встроенном репозитории
2. **Chat**: Интерактивное общение с моделью
3. **Local Server**: Запуск REST API (как Ollama)

**Преимущества:**
- ✅ Визуальный интерфейс для настройки параметров (temperature, top_p)
- ✅ Встроенный поиск моделей от разных провайдеров
- ✅ Мониторинг использования ресурсов (RAM, GPU)
- ✅ API сервер совместим с Ollama

---

#### llama.cpp — максимальная производительность

**Что это:** C++ библиотека для инференса LLM с оптимизацией под CPU/GPU

**Установка через Homebrew (macOS):**
```bash
brew install llama.cpp
```

**Установка из исходников:**
```bash
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
make

# С GPU поддержкой (CUDA для NVIDIA, Metal для Mac)
make LLAMA_CUBLAS=1   # NVIDIA CUDA
make LLAMA_METAL=1    # Apple Metal (macOS)
```

**Запуск модели:**
```bash
# Скачать GGUF модель (пример: Llama 3.1 8B Q4_K_M)
# Из huggingface.co или llama.cpp репозитория

./main -m models/llama-3.1-8b.Q4_K_M.gguf \
       -p "Напиши Kotlin функцию для сортировки списка" \
       -n 512 \
       --temp 0.7

# Интерактивный режим:
./main -m models/llama-3.1-8b.Q4_K_M.gguf -interactive
```

**Конвертация моделей в GGUF:**
```bash
# Из PyTorch weights в GGUF
python convert-hf-to-gguf.py models/llama-3.1-8b \
       --outfile models/llama-3.1-8b-f16.gguf

# Квантование
./quantize models/llama-3.1-8b-f16.gguf \
           models/llama-3.1-8b.Q4_K_M.gguf \
           Q4_K_M
```

---

#### Text Generation WebUI — полноценный сервер

**Установка:**
```bash
# Клонирование репозитория
git clone https://github.com/oobabooga/text-generation-webui
cd text-generation-webui

# Установка зависимостей
pip install -r requirements.txt

# Запуск
python server.py
```

**Возможности:**
- 🎨 Веб-интерфейс с чатом и настройками
- 🔌 Поддержка множества моделей (GGUF, Safetensors)
- 🧩 Экстенсивные плагины (API, TTS, изображения)
- ⚙️ Детальная настройка параметров генерации

---

### Тема 9.3: Интеграция локальных моделей в KMP проект (8 часов)

#### Архитектура интеграции

```
┌─────────────────┐     HTTP Request      ┌──────────────┐
│   KMP App       │ ←──────────────────→  │ Ollama/LM    │
│ (Android/iOS/   │     localhost:11434   │ Server       │
│  Desktop)       │                       └──────────────┘
└─────────────────┘
```

#### Реализация через expect/actual

**commonMain — интерфейс:**
```kotlin
interface LocalLLMService {
    suspend fun generate(prompt: String): LLMResponse
    suspend fun chat(messages: List<ChatMessage>): ChatResponse
    
    fun isAvailable(): Boolean
    fun getModelInfo(): ModelInfo?
}

data class LLMResponse(
    val content: String,
    val model: String,
    val tokensUsed: Int,
    val generationTime: Long // ms
)

data class ChatMessage(
    val role: MessageRole, // user, assistant, system
    val content: String
)

data class ChatResponse(
    val message: ChatMessage,
    val model: String,
    val tokensUsed: Int
)

data class ModelInfo(
    val name: String,
    val size: Long, // bytes
    val parameters: String, // "8B", "70B"
    val quantization: String // "Q4_K_M"
)

enum class MessageRole {
    USER, ASSISTANT, SYSTEM
}
```

**androidMain — реализация:**
```kotlin
import io.ktor.client.*
import io.ktor.client.call.*
import io.ktor.client.engine.cio.*
import io.ktor.client.request.*
import io.ktor.http.*

class LocalLLMServiceImpl : LocalLLMService {
    private val client = HttpClient(CIO)
    
    override suspend fun generate(prompt: String): LLMResponse {
        return try {
            val response = client.post("http://localhost:11434/api/generate") {
                contentType(ContentType.Application.Json)
                setBody("""{
                    "model": "llama3.1",
                    "prompt": "$prompt",
                    "stream": false
                }""")
            }.body<GenerateResponse>()
            
            LLMResponse(
                content = response.response,
                model = response.model,
                tokensUsed = estimateTokens(response.response),
                generationTime = System.currentTimeMillis() - startTime
            )
        } catch (e: Exception) {
            throw LLMException("Failed to generate: ${e.message}")
        }
    }
    
    override suspend fun chat(messages: List<ChatMessage>): ChatResponse {
        return try {
            val response = client.post("http://localhost:11434/api/chat") {
                contentType(ContentType.Application.Json)
                setBody("""{
                    "model": "llama3.1",
                    "messages": ${serializeMessages(messages)}
                }""")
            }.body<ChatApiResponse>()
            
            ChatResponse(
                message = ChatMessage(
                    role = MessageRole.ASSISTANT,
                    content = response.message.content
                ),
                model = response.model,
                tokensUsed = estimateTokens(response.message.content)
            )
        } catch (e: Exception) {
            throw LLMException("Failed to chat: ${e.message}")
        }
    }
    
    override fun isAvailable(): Boolean {
        return try {
            val socket = Socket("localhost", 11434)
            socket.close()
            true
        } catch (e: Exception) {
            false
        }
    }
    
    override fun getModelInfo(): ModelInfo? {
        // Android-specific: можно кэшировать информацию о модели
        return null // TODO: реализовать
    }
}

// Data классы для Ollama API
data class GenerateResponse(
    val model: String,
    val response: String,
    val done: Boolean
)

data class ChatApiResponse(
    val model: String,
    val message: ApiMessage
)

data class ApiMessage(
    val role: String,
    val content: String
)
```

**iosMain — реализация:**
```kotlin
import platform.Foundation.NSURL
import platform.Foundation.NSTemporaryDirectory
// iOS-specific implementation using URLSession

class LocalLLMServiceImpl : LocalLLMService {
    // Аналогичная реализация с URLSession вместо Ktor Client
    // Или использование нативного Network framework
    
    override suspend fun generate(prompt: String): LLMResponse {
        // iOS-specific HTTP client implementation
    }
    
    override suspend fun chat(messages: List<ChatMessage>): ChatResponse {
        // iOS-specific implementation
    }
    
    override fun isAvailable(): Boolean {
        // Проверка доступности локального сервера
    }
    
    override fun getModelInfo(): ModelInfo? {
        // iOS-specific model info retrieval
    }
}
```

**desktopMain — реализация:**
```kotlin
// Desktop может использовать прямой вызов llama.cpp через JNI
// Или HTTP клиент как Android

class LocalLLMServiceImpl : LocalLLMService {
    // Desktop-specific implementation
    // Можно запускать llama.cpp как subprocess для максимальной производительности
    
    override suspend fun generate(prompt: String): LLMResponse {
        // Вариант 1: HTTP запрос к Ollama
        // Вариант 2: Прямой вызов llama.cpp через ProcessBuilder
        
        val process = ProcessBuilder(
            "./llama/main",
            "-m", "models/llama-3.1-8b.Q4_K_M.gguf",
            "-p", prompt,
            "-n", "512"
        ).start()
        
        val output = process.inputStream.bufferedReader().readText()
        
        return LLMResponse(
            content = output,
            model = "llama-3.1-8b",
            tokensUsed = estimateTokens(output),
            generationTime = System.currentTimeMillis() - startTime
        )
    }
    
    // Остальные методы...
}
```

---

### Тема 9.4: Оптимизация локальных моделей (6 часов)

#### GPU Acceleration

**NVIDIA CUDA:**
```bash
# Установка CUDA toolkit
# Затем компиляция llama.cpp с поддержкой CUDA

make LLAMA_CUBLAS=1

# Запуск с GPU
./main -m model.gguf --n-gpu-layers 99 -p "prompt"
```

**Apple Metal (macOS):**
```bash
# Автоматически поддерживается в llama.cpp на macOS

./main -m model.gguf --n-gpu-layers 99 -p "prompt"
```

**Vulkan (Linux/Windows):**
```bash
make LLAMA_VULKAN=1

./main -m model.gguf --n-gpu-layers 99 -p "prompt"
```

#### Context Window Management

**Проблема:** Локальные модели имеют ограниченное контекстное окно (8K-32K токенов)

**Решения:**

1. **Sliding Window** — отбрасывание старых сообщений
```kotlin
class SlidingWindowContextManager(
    private val maxTokens: Int = 8000,
) {
    private val messages = mutableListOf<ChatMessage>()
    
    fun addMessage(message: ChatMessage) {
        messages.add(message)
        
        val currentTokens = messages.sumOf { estimateTokens(it.content) }
        if (currentTokens > maxTokens) {
            trimToMaxTokens()
        }
    }
    
    private fun trimToMaxTokens() {
        // Удаляем старые сообщения, сохраняя последние
        while (messages.sumOf { estimateTokens(it.content) } > maxTokens) {
            // Удаляем самое старое сообщение пользователя-ассистента
            val indexToRemove = findOldestUserAssistantPair()
            messages.removeAt(indexToRemove)
            messages.removeAt(indexToRemove + 1)
        }
    }
}
```

2. **Summary Compression** — сжатие старых сообщений в summary
```kotlin
class SummaryContextManager(
    private val llm: LocalLLMService,
) {
    suspend fun compressOldMessages(messages: List<ChatMessage>): List<ChatMessage> {
        val oldMessages = messages.take(messages.size - 4) // Сохраняем последние 4
        val recentMessages = messages.takeLast(4)
        
        // Генерируем summary старых сообщений
        val summaryPrompt = """
            Сожми следующие сообщения в краткое резюме (максимум 200 символов):
            ${oldMessages.joinToString("\n") { "${it.role}: ${it.content}" }}
        """
        
        val summary = llm.generate(summaryPrompt)
        
        return listOf(
            ChatMessage(MessageRole.SYSTEM, summary.content),
        ) + recentMessages
    }
}
```

3. **RAG (Retrieval Augmented Generation)** — поиск релевантной информации
```kotlin
class RAGContextManager(
    private val vectorStore: VectorStore,
    private val llm: LocalLLMService,
) {
    suspend fun generateWithRAG(query: String): LLMResponse {
        // Поиск релевантных документов
        val relevantDocs = vectorStore.search(query, topK = 3)
        
        // Формирование промпта с контекстом
        val contextPrompt = """
            Используя следующую информацию, ответь на вопрос:
            
            Контекст:
            ${relevantDocs.joinToString("\n\n")}
            
            Вопрос: $query
            
            Ответ:
        """
        
        return llm.generate(contextPrompt)
    }
}
```

#### Prompt Caching

**Проблема:** Повторные запросы с одинаковым промптом тратят время

**Решение:** Кэширование результатов
```kotlin
class CachedLLMService(
    private val delegate: LocalLLMService,
    private val cache: MutableMap<String, LLMResponse> = mutableMapOf(),
) : LocalLLMService by delegate {
    
    override suspend fun generate(prompt: String): LLMResponse {
        val cacheKey = normalizePrompt(prompt)
        
        return cache.getOrPut(cacheKey) {
            delegate.generate(prompt).also { response ->
                // Асинхронное сохранение в persistent cache
                saveToPersistentCache(cacheKey, response)
            }
        }
    }
    
    private fun normalizePrompt(prompt: String): String {
        return prompt.lowercase().replace(Regex("\\s+"), " ").trim()
    }
}
```

---

### Тема 9.5: Конфиденциальность и гибридный подход (4 часа)

#### Преимущества локальных моделей для конфиденциальности:

1. **Финансовые данные** — транзакции, балансы не покидают устройство
2. **Медицинская информация** — health data остается приватной
3. **Бизнес-логика** — proprietary алгоритмы не раскрываются
4. **Персональные данные** — PII (Personally Identifiable Information) защищена

#### Гибридный подход: когда что использовать?

```kotlin
enum class LLMProvider {
    LOCAL, CLOUD, HYBRID
}

class HybridLLMService(
    private val local: LocalLLMService,
    private val cloud: CloudLLMService, // OpenAI, Claude API
) : LocalLLMService {
    
    override suspend fun generate(prompt: String): LLMResponse {
        // Анализ промпта для выбора провайдера
        val provider = selectProvider(prompt)
        
        return when (provider) {
            LLMProvider.LOCAL -> local.generate(prompt)
            LLMProvider.CLOUD -> cloud.generate(prompt)
            LLMProvider.HYBRID -> {
                // Локальная модель для черновика, cloud для refinement
                val draft = local.generate(prompt)
                cloud.refine(draft.content)
            }
        }
    }
    
    private fun selectProvider(prompt: String): LLMProvider {
        // Правила выбора провайдера:
        
        // 1. Конфиденциальные данные → LOCAL
        if (containsSensitiveData(prompt)) return LLMProvider.LOCAL
        
        // 2. Сложные задачи → CLOUD
        if (isComplexTask(prompt)) return LLMProvider.CLOUD
        
        // 3. Рутинные задачи → LOCAL
        if (isRoutineTask(prompt)) return LLMProvider.LOCAL
        
        // 4. Критичное качество → HYBRID
        if (isQualityCritical(prompt)) return LLMProvider.HYBRID
        
        // 5. По умолчанию → LOCAL
        return LLMProvider.LOCAL
    }
    
    private fun containsSensitiveData(prompt: String): Boolean {
        // Проверка на PII, финансовые данные, health info
        return prompt.contains(Regex("\\b\\d{16}\\b")) // Credit card pattern
    }
    
    private fun isComplexTask(prompt: String): Boolean {
        // Сложные задачи требуют cloud моделей с большим контекстом
        return prompt.contains("анализируй") || 
               prompt.contains("спроектируй архитектуру")
    }
}
```

---

## 📝 Практические задания

### Задание 9.1: Запустить Ollama с Llama 3 и интегрировать в KMP проект (4 часа)

**Цель:** Настроить локальную LLM и сделать первый запрос из KMP приложения

**Шаги:**
1. Установить Ollama и скачать llama3.1
2. Создать expect/actual интерфейс LocalLLMService в KMP проекте
3. Реализовать androidMain с Ktor Client
4. Протестировать генерацию простого Kotlin кода

**Критерии успеха:**
- ✓ Ollama запущен и доступен на localhost:11434
- ✓ KMP приложение успешно делает запрос к локальной модели
- ✓ Генерируемый код компилируется

---

### Задание 9.2: Создать expect/actual для вызова локальной LLM на Android/iOS/Desktop (6 часов)

**Цель:** Полная кроссплатформенная реализация LocalLLMService

**Шаги:**
1. Расширить интерфейс в commonMain (generate, chat, isAvailable, getModelInfo)
2. Реализовать androidMain с Ktor Client + CIO engine
3. Реализовать iosMain с URLSession или нативным Network framework
4. Реализовать desktopMain с прямым вызовом llama.cpp через ProcessBuilder

**Критерии успеха:**
- ✓ Работает на всех 3 платформах
- ✓ Обработка ошибок (сервер не запущен, таймаут)
- ✓ Кэширование результатов для повторяющихся запросов

---

### Задание 9.3: Настроить квантованную модель (4-5GB) для работы на ноутбуке (4 часа)

**Цель:** Оптимизация использования памяти и производительности

**Шаги:**
1. Скачать Llama 3.1 8B в формате Q4_K_M (~4.7GB)
2. Настроить Ollama/LM Studio для использования GPU (если доступно)
3. Протестировать скорость генерации (токенов/секунду)
4. Сравнить с Q5_K_M и Q8_0 версиями

**Критерии успеха:**
- ✓ Модель загружается в память (не OOM)
- ✓ Скорость генерации ≥ 10 токенов/сек на CPU
- ✓ Скорость генерации ≥ 50 токенов/сек на GPU

---

## 📊 Проверка знаний

### Quiz (10 вопросов)

1. **Какой формат квантования рекомендуется для баланса качество/размер?**
   - a) Q4_K_M ✓
   - b) Q8_0
   - c) FP16

2. **Какой инструмент самый простой для запуска локальных моделей?**
   - a) llama.cpp
   - b) Ollama ✓
   - c) Text Generation WebUI

3. **Какой размер контекста у большинства локальных моделей?**
   - a) 8K-32K токенов ✓
   - b) 100K+ токенов
   - c) 4K токенов

4. **Что такое GGUF формат?**
   - a) Формат для квантованных моделей llama.cpp ✓
   - b) Формат изображений
   - c) Протокол передачи данных

5. **Какой подход лучше для конфиденциальных данных?**
   - a) Cloud LLM
   - b) Локальная модель ✓
   - c) Гибридный

(и ещё 5 вопросов...)

---

## 🎯 Итоговый проект модуля

**Задача:** Создать KMP приложение с локальной LLM для конфиденциальных задач

**Требования:**
1. Полная expect/actual реализация для Android/iOS/Desktop
2. Поддержка Ollama и llama.cpp
3. Кэширование результатов
4. Sliding window context management
5. Документация по настройке на разных платформах

**Критерии оценки:**
- ✓ Приложение работает оффлайн (без интернета)
- ✓ Поддерживает чат с историей сообщений
- ✓ Обработка ошибок и fallback на cloud (опционально)

---

## 📚 Дополнительные материалы

### Документация:
- [Ollama Documentation](https://github.com/ollama/ollama)
- [llama.cpp Documentation](https://github.com/ggerganov/llama.cpp)
- [LM Studio Guide](https://lmstudio.ai/)

### Репозитории моделей:
- [Hugging Face GGUF Models](https://huggingface.co/models?library=gguf)
- [Ollama Library](https://ollama.com/library)

### Статьи:
- "Running Llama 3 Locally" — подробный гайд
- "Quantization Explained" — что такое Q4_K_M, Q5_K_M

### Видео:
- "Ollama Tutorial for Developers" (YouTube, 30 мин)
- "llama.cpp Deep Dive" (YouTube, 45 мин)

---

## 🚀 Следующий шаг

Поздравляю! Вы завершили AI Agent Development Track! 🎉

**Итоговый проект:** Создайте приложение с multi-agent системой и локальной LLM

---

**Удачи в работе с локальными моделями! 🔒🤖**

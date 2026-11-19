<h1>Модуль 1: Введение в Kotlin Multiplatform и настройка окружения</h1>

<hr/>

<h2>С чего всё начинается</h2>

<p>Представь ситуацию: ты разработчик в продуктовой команде. Пришло ТЗ — сделать приложение для трекинга привычек. Дедлайн через месяц.</p>

<p>И тут выясняется, что нужно <strong>два</strong> приложения:</p>

<ul>
    <li>Android-версия — на Kotlin + Compose</li>
    <li>iOS-версия — на Swift + SwiftUI</li>
</ul>

<p>Два разработчика. Два стека. Две базы данных. Два набора багов. И каждый раз, когда меняется бизнес-логика (а она меняется), нужно вносить правку <strong>дважды</strong>. А ещё лучше — дважды с разными ошибками, потому что один разработчик что-то упустил.</p>

<p>Знакомая боль?</p>

<p><strong>Kotlin Multiplatform (KMP)</strong> — это ответ на эту боль. Одна кодовая база для бизнес-логики, которая компилируется и на Android, и на iOS, и даже на Desktop.</p>

<p>Но давай не будем сразу прыгать в омут. Сначала разберёмся, <strong>как именно</strong> KMP решает эту проблему и <strong>что из кода</strong> можно переиспользовать, а <strong>что</strong> — нет.</p>

<hr/>

<h2>Что мы будем делать в этом модуле</h2>

<table>
    <thead>
        <tr>
            <th>#</th>
            <th>Задача</th>
            <th>Время</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>1</td>
            <td>Понять, как устроен KMP-проект изнутри</td>
            <td>~30 мин</td>
        </tr>
        <tr>
            <td>2</td>
            <td>Разобраться с <code>expect/actual</code> на пальцах</td>
            <td>~30 мин</td>
        </tr>
        <tr>
            <td>3</td>
            <td>Написать первый экран, который работает на двух платформах</td>
            <td>~1 час</td>
        </tr>
        <tr>
            <td>4</td>
            <td>Выполнить домашнее задание</td>
            <td>~2 часа</td>
        </tr>
    </tbody>
</table>

<p><strong>Итого:</strong> 5–7 часов. Можно растянуть на два-три вечера.</p>

<blockquote>
<p>💡 <strong>Важно:</strong> Для iOS-части нужен Mac с Xcode. Если у вас Windows — не паникуйте. Вы всё равно сможете писать общий код, запускать на Android, а iOS-часть просто не запускаете локально. Мы на это учтём.</p>
</blockquote>

<hr/>

<h2>1. Архитектура: один код — много платформ</h2>

<h3>Аналогия из жизни</h3>

<p>Думай о KMP-проекте как о <strong>ресторане с международной франшизой</strong>:</p>

<pre class="mermaid">
flowchart TD
    Common["<div style='width: 340px; text-align: left;'><b>🏢 Корпоративный офис (commonMain)</b><br/><br/>• Рецепты блюд (бизнес-логика)<br/>• Стандарты сервиса (UI-компоненты)<br/>• Бухгалтерия (модели данных)</div>"]
    Moscow["<div style='width: 290px; text-align: left;'><b>🇷🇺 Москва (androidMain)</b><br/><br/>Повар-русский готовит по тем же рецептам, но использует свои сковородки и плиту</div>"]
    Tokyo["<div style='width: 290px; text-align: left;'><b>🇯🇵 Токио (iosMain)</b><br/><br/>Повар-японский готовит по тем же рецептам, но использует свои ножи и духовку</div>"]

    Common --> Moscow
    Common --> Tokyo

    style Common fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#000
    style Moscow fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#000
    style Tokyo fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:#000
</pre>

<p><strong>Корпоративный офис</strong> (<code>commonMain</code>) — это вся логика, которая <strong>не зависит</strong> от платформы: расчёты, бизнес-правила, модели данных, UI-компоненты на Compose.</p>

<p><strong>Филиалы</strong> (<code>androidMain</code>, <code>iosMain</code>) — это «склейка» с платформой: запуск активности, доступ к камере, push-уведомления, работа с файловой системой.</p>

<h3>Теперь посмотри на дерево папок</h3>

<p>Когда ты создашь KMP-проект, структура будет такой:</p>

<pre>
<code>MyProject/
├── shared/                          ← общий модуль (shared module)
│   └── src/
│       ├── commonMain/kotlin/       ← 🔥 80-90% кода живёт здесь
│       │   ├── model/               ← данные, сущности
│       │   ├── logic/               ← бизнес-логика
│       │   ├── ui/                  ← Compose-экраны
│       │   └── platform/            ← expect-объявления
│       │
│       ├── commonTest/kotlin/       ← тесты общего кода
│       │
│       ├── androidMain/kotlin/      ← 🤖 что-то Android-specific
│       │   └── platform/            ← actual для Android
│       │
│       ├── iosMain/kotlin/          ← 🍎 что-то iOS-specific
│       │   └── platform/            ← actual для iOS
│       │
│       └── desktopMain/kotlin/      ← 💻 (опционально)
│
├── androidApp/                      ← Android-приложение (нативная обёртка)
│   └── src/main/kotlin/
│       └── MainActivity.kt          ← точка входа для Android
│
└── iosApp/                          ← Xcode-проект (нативная обёртка)
    └── iOSApp/
        └── ContentView.swift        ← точка входа для iOS</code></pre>

<h3>Ключевой вопрос: что именно можно положить в commonMain?</h3>

<p>Вот чек-лист. Отметь, что для тебя очевидно, а что — нет:</p>

<table>
    <thead>
        <tr>
            <th>Что</th>
            <th>В commonMain?</th>
            <th>Почему</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Модели данных (data class)</td>
            <td>✅</td>
            <td>Чистый Kotlin, не зависит от ОС</td>
        </tr>
        <tr>
            <td>Бизнес-логика (расчёты, валидация)</td>
            <td>✅</td>
            <td>Чистый Kotlin</td>
        </tr>
        <tr>
            <td>UI на Compose Multiplatform</td>
            <td>✅</td>
            <td>Компилируется под все платформы</td>
        </tr>
        <tr>
            <td>Coroutines, Flow</td>
            <td>✅</td>
            <td>Кроссплатформенные</td>
        </tr>
        <tr>
            <td>SQLDelight (БД)</td>
            <td>✅</td>
            <td>Есть драйверы для всех платформ</td>
        </tr>
        <tr>
            <td>Ktor Client (сеть)</td>
            <td>✅</td>
            <td>Есть движки для всех платформ</td>
        </tr>
        <tr>
            <td><code>android.util.Log</code></td>
            <td>❌</td>
            <td>Это Android API</td>
        </tr>
        <tr>
            <td><code>NSFileManager</code></td>
            <td>❌</td>
            <td>Это iOS API</td>
        </tr>
        <tr>
            <td>Доступ к камере</td>
            <td>❌</td>
            <td>Разные API → через expect/actual</td>
        </tr>
        <tr>
            <td>Push-уведомления</td>
            <td>❌</td>
            <td>FCM vs APNs → через expect/actual</td>
        </tr>
        <tr>
            <td>Работа с файловой системой</td>
            <td>❌</td>
            <td>Разные пути → через expect/actual</td>
        </tr>
    </tbody>
</table>

<blockquote>
<p>📌 <strong>Правило:</strong> Если ты используешь только <code>kotlin.*</code>, корутины, Compose, SQLDelight, Ktor — ты в <code>commonMain</code>. Нужен что-то из ОС? → <code>expect/actual</code>.</p>
</blockquote>

<hr/>

<h2>2. Expect/Actual: как одна функция живёт на двух платформах</h2>

<h3>Проблема</h3>

<p>Ты пишешь в <code>commonMain</code> функцию «получить имя платформы». На Android ты хочешь получить <code>"Android"</code>, на iOS — <code>"iOS"</code>. Но API у них разные. Как быть?</p>

<h3>Решение: объявляю контракт, а реализую отдельно</h3>

<p>Думай об <code>expect</code> как о <strong>заказе в ресторане</strong>: ты говоришь «дайте мне напиток» (объявление), а бармен на каждой станции (платформа) льёт свой (реализация).</p>

<pre class="mermaid">
flowchart TD
    Expect["<div style='width: 360px; text-align: left;'><b>commonMain (объявление)</b><br/><br/><code>expect fun getPlatformName(): String</code><br/><br/>«я знаю, что такая функция есть,<br/>но не знаю как она работает»</div>"]
    Android["<div style='width: 260px; text-align: left;'><b>androidMain (реализация)</b><br/><br/><code>actual fun getPlatformName(): String<br/>= "Android"</code></div>"]
    IOS["<div style='width: 260px; text-align: left;'><b>iosMain (реализация)</b><br/><br/><code>actual fun getPlatformName(): String<br/>= "iOS"</code></div>"]

    Expect --> Android
    Expect --> IOS

    style Expect fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#000
    style Android fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#000
    style IOS fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:#000
</pre>

<h3>Код</h3>

<p>Вот как это выглядит. Три файла, три источника:</p>

<p><strong>Файл 1:</strong> <code>shared/src/commonMain/kotlin/platform/Platform.kt</code></p>

<pre>
<code class="language-kotlin">package com.ecotrack.platform

// Объявляем: «такая функция существует, но я не знаю, как она работает»
expect fun getPlatformName(): String</code></pre>

<p><strong>Файл 2:</strong> <code>shared/src/androidMain/kotlin/platform/Platform.kt</code></p>

<pre>
<code class="language-kotlin">package com.ecotrack.platform

// Реализация для Android
actual fun getPlatformName(): String = "Android"</code></pre>

<p><strong>Файл 3:</strong> <code>shared/src/iosMain/kotlin/platform/Platform.kt</code></p>

<pre>
<code class="language-kotlin">package com.ecotrack.platform

// Реализация для iOS
actual fun getPlatformName(): String = "iOS"</code></pre>

<p><strong>И в любом месте</strong> <code>commonMain</code> <strong>ты просто вызываешь:</strong></p>

<pre>
<code class="language-kotlin">val greeting = "Привет, ${getPlatformName()}!"
// На Android: "Привет, Android!"
// На iOS:     "Привет, iOS!"</code></pre>

<h3>Подводные камни</h3>

<blockquote>
<p>⚠️ <strong>Пакеты и имена должны совпадать точно.</strong> <code>expect</code> в <code>commonMain</code> и <code>actual</code> в <code>androidMain</code> должны быть в одном и том же пакете, с тем же именем. Иначе компилятор выдаст:</p>

<pre>
<code>e: ACTUAL without corresponding EXPECT</code></pre>
</blockquote>

<blockquote>
<p>⚠️ <strong><code>expect</code> — это не интерфейс.</strong> Ты не можешь объявить <code>expect class</code> с методом и потом реализовать его как хочешь. <code>expect</code> — это обещание: «в каждой платформе будет соответствующий <code>actual</code>». Для более сложных случаев (разные методы в разных платформах) используй <strong>platform-specific sealed interfaces</strong> или <strong>Service Locator pattern</strong> — разберём в Модуле 7.</p>
</blockquote>

<h3>Когда expect/actual НЕ нужен</h3>

<p>Частая ошибка новичков — оборачивать в <code>expect/actual</code> то, что не зависит от платформы:</p>

<pre>
<code class="language-kotlin">// ❌ НЕ надо так
expect fun parseInt(s: String): Int  // это есть в stdlib, работает везде

// ✅ Просто используй
val x = "42".toInt()  // работает на всех платформах из коробки</code></pre>

<hr/>

<h2>3. Создаём проект EcoTrack</h2>

<h3>Как подойти к решению</h3>

<p>Прежде чем нажимать кнопки в IDE, давай разберёмся, <strong>что именно</strong> мы делаем и <strong>зачем</strong> каждый шаг:</p>

<ol>
    <li><strong>Выбираем wizard</strong> — потому что создавать KMP-проект вручную (писать Gradle-скрипты, подключать плагины) — это боль. Wizard делает это за нас.</li>
    <li><strong>Выбираем target'ы</strong> — это те платформы, под которые мы компилируем общий код.</li>
    <li><strong>Выбираем UI framework</strong> — Compose Multiplatform даст нам декларативный UI на Kotlin.</li>
    <li><strong>Смотрим на build.gradle.kts</strong> — чтобы понять, что именно настроено под капотом.</li>
</ol>

<h3>Пошагово</h3>

<p><strong>Шаг 1: Открой Android Studio (или IntelliJ IDEA)</strong></p>

<p>Минимальные требования:</p>

<ul>
    <li>JDK 17+ (встроенная в Android Studio подойдёт)</li>
    <li>Последний плагин <strong>Kotlin Multiplatform</strong> (устанавливается из Plugin Repository)</li>
</ul>

<p><strong>Шаг 2: Создай проект</strong></p>

<pre>
<code>File → New → New Project → Multiplatform Mobile (или Multiplatform Application)</code></pre>

<p><strong>Шаг 3: Настрой параметры</strong></p>

<table>
    <thead>
        <tr>
            <th>Параметр</th>
            <th>Значение</th>
            <th>Зачем</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Project name</td>
            <td>EcoTrack</td>
            <td>Имя проекта</td>
        </tr>
        <tr>
            <td>Target</td>
            <td>Android, iOS</td>
            <td>Компилируем под обе</td>
        </tr>
        <tr>
            <td>UI Framework</td>
            <td>Compose Multiplatform</td>
            <td>Declarative UI</td>
        </tr>
        <tr>
            <td>Kotlin version</td>
            <td>последняя стабильная</td>
            <td>Не будем джиттерить версии</td>
        </tr>
    </tbody>
</table>

<p>Нажми <strong>Finish</strong>. Подожди, пока Gradle скачает зависимости (первый раз может занять 5-10 минут).</p>

<p><strong>Шаг 4: Изучи <code>shared/build.gradle.kts</code></strong></p>

<p>Открой файл. Вот ключевые секции:</p>

<pre>
<code class="language-kotlin">plugins {
    kotlin("multiplatform")
    kotlin("plugin.compose")          // ← Compose Multiplatform
    kotlin("plugin.serialization")    // ← для JSON (разберём в Модуле 5)
}

kotlin {
    // ─── Целевые платформы ───
    androidTarget {
        // Настройки для Android
    }
    
    iosX64()       // ← симулятор на Intel Mac
    iosArm64()     // ← реальный iPhone (Apple Silicon)
    iosSimulatorArm64() // ← симулятор на Apple Silicon (M1/M2/M3)
    
    // ─── Зависимости, общие для всех платформ ───
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation(compose.runtime)
                implementation(compose.foundation)
                implementation(compose.material3)
                // ...
            }
        }
    }
}</code></pre>

<blockquote>
<p>📌 <strong>Запомни:</strong> всё, что в <code>commonMain</code> dependencies &mdash; доступно на всех платформах. Если нужна зависимость только для Android &mdash; ищи секцию <code>androidMain</code>.</p>
</blockquote>

<details><summary>🔍 Развернуть: полный build.gradle.kts (чтобы сверить с твоим)</summary>

<pre>
<code class="language-kotlin">plugins {
    kotlin("multiplatform") version "2.0.20"
    kotlin("plugin.compose") version "2.0.20"
    kotlin("plugin.serialization") version "2.0.20"
    id("com.android.library") version "8.3.0"
}

kotlin {
    androidTarget {
        compilations.all {
            kotlinOptions {
                jvmTarget = "17"
            }
        }
    }
    
    iosX64()
    iosArm64()
    iosSimulatorArm64()
    
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.compose.runtime:runtime:1.6.2")
                implementation("org.jetbrains.compose.foundation:foundation:1.6.2")
                implementation("org.jetbrains.compose.material3:material3:1.6.2")
                implementation("org.jetbrains.compose.ui:ui:1.6.2")
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
                implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")
            }
        }
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
            }
        }
        val androidMain by getting
        val iosMain by getting
    }
}

android {
    namespace = "com.ecotrack.shared"
    compileSdk = 34
    defaultConfig {
        minSdk = 24
    }
}

compose.runtime {
    entryPoints = listOf("androidx.compose.ui.window.Recomposer")
}</code></pre>

<blockquote>
<p>⚠️ Версии могут отличаться в зависимости от того, когда ты создаёшь проект. Не копируй вслепую &mdash; смотри, что сгенерировал wizard.</p>
</blockquote>

</details>

<hr/>

<h2>4. Первый экран: от идеи до запуска</h2>

<h3>Как подойти к решению</h3>

<p>Мы хотим написать <strong>один</strong> Compose-экран, который будет:</p>

<ul>
    <li>Показывать заголовок</li>
    <li>Считать клики по кнопке</li>
    <li>Работать и на Android, и на iOS</li>
</ul>

<p>Давай разложим на шаги:</p>

<pre class="mermaid">
flowchart TD
    Screen["<div style='width: 380px; text-align: left;'><b>MainScreen() — commonMain</b><br/><br/>• Text("Добро пожаловать в EcoTrack!")<br/>• Text("Привычек добавлено: $count")<br/>• Button("Добавить") { count++ }</div>"]
    Android["<div style='width: 280px; text-align: left;'><b>androidMain</b><br/><br/>MainActivity<br/><code>setContent { MainScreen() }</code></div>"]
    IOS["<div style='width: 280px; text-align: left;'><b>iosMain</b><br/><br/>MainViewController<br/><code>ComposeUIViewController { MainScreen() }</code></div>"]

    Screen --> Android
    Screen --> IOS

    style Screen fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#000
    style Android fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#000
    style IOS fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:#000
</pre>

<p>Видишь? <strong>Один и тот же</strong> <code>MainScreen()</code> вставляется в две разные «обёртки». Android-обёртка — это <code>ComponentActivity</code> с <code>setContent</code>. iOS-обёртка — это <code>ComposeUIViewController</code>. А сам экран — один, общий.</p>

<h3>Пишем код</h3>

<p><strong>Шаг 1: Общий экран</strong></p>

<p>Создай файл <code>shared/src/commonMain/kotlin/ui/MainScreen.kt</code>:</p>

<details><summary>🔍 Развернуть: код MainScreen.kt</summary>

<pre>
<code class="language-kotlin">package com.ecotrack.ui

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.height
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableIntStateOf
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import com.ecotrack.platform.getPlatformName

@Composable
fun MainScreen() {
    var count by mutableIntStateOf(0)

    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            text = "Добро пожаловать в EcoTrack!",
            style = MaterialTheme.typography.headlineMedium
        )

        Spacer(modifier = Modifier.height(16.dp))

        Text(
            text = getPlatformName(),
            style = MaterialTheme.typography.bodyMedium
        )

        Spacer(modifier = Modifier.height(16.dp))

        Text(text = "Привычек добавлено: $count")

        Spacer(modifier = Modifier.height(24.dp))

        Button(onClick = { count++ }) {
            Text("Добавить привычку")
        }
    }
}</code></pre>

<blockquote>
<p>💡 <strong>Примечание:</strong> Здесь мы уже используем <code>getPlatformName()</code> из модуля <code>platform</code> — тот самый <code>expect/actual</code>, который мы разбирали выше. Так что этот экран уже демонстрирует кроссплатформенность: на Android будет написано "Android", на iOS — "iOS".</p>
</blockquote>

</details>

<p><strong>Шаг 2: Android-обёртка</strong></p>

<p>Открой <code>androidApp/src/main/kotlin/MainActivity.kt</code>:</p>

<details><summary>🔍 Развернуть: код MainActivity.kt (Android)</summary>

<pre>
<code class="language-kotlin">package com.ecotrack

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.ui.Modifier
import com.ecotrack.ui.MainScreen

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    MainScreen()
                }
            }
        }
    }
}</code></pre>

<blockquote>
<p>📌 <strong>Что здесь происходит:</strong> <code>ComponentActivity</code> — это стандартная Android-активность. Метод <code>setContent { }</code> — это мост между Activity и Compose. Внутри мы вызываем <strong>тот же самый</strong> <code>MainScreen()</code> из <code>commonMain</code>. Всё.</p>
</blockquote>

</details>

<p><strong>Шаг 3: iOS-обёртка</strong></p>

<p>В iOS-части Xcode-проекта есть Swift-файл, который создаёт <code>UIViewController</code>. Но с Compose Multiplatform мы делаем это проще — через Kotlin:</p>

<p>Создай <code>shared/src/iosMain/kotlin/MainViewController.kt</code>:</p>

<details><summary>🔍 Развернуть: код MainViewController.kt (iOS)</summary>

<pre>
<code class="language-kotlin">package com.ecotrack

import androidx.compose.ui.window.ComposeUIViewController
import com.ecotrack.ui.MainScreen

fun MainViewController() = ComposeUIViewController {
    MainScreen()
}</code></pre>

<p>Вот и всё. Одна строка. <code>ComposeUIViewController</code> — это обёртка, которая создаёт <code>UIViewController</code>, внутри которого рендерится наше Compose-дерево.</p>

</details>

<p>Затем в Swift-коде Xcode-проекта (<code>iosApp/iOSApp/ContentView.swift</code>):</p>

<pre>
<code class="language-swift">import SwiftUI
import shared

struct ContentView: View {
    var body: some View {
        MainViewController()
            .ignoresSafeArea()
    }
}</code></pre>

<blockquote>
<p>📌 <code>import shared</code> — это импорт Kotlin-модуля в Swift. KMP экспортирует Swift-фреймворк из commonMain/iosMain, и Xcode видит его как обычный Swift-модуль.</p>
</blockquote>

<p><strong>Шаг 4: Запуск</strong></p>

<table>
    <thead>
        <tr>
            <th>Платформа</th>
            <th>Как запустить</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Android</td>
            <td>Выбери эмулятор в run-конфигурации → ▶️</td>
        </tr>
        <tr>
            <td>iOS</td>
            <td>Выбери iOS-симулятор → ▶️ (нужен Mac)</td>
        </tr>
    </tbody>
</table>

<p><strong>Ожидаемый результат:</strong></p>

<pre class="mermaid">
flowchart TD
    Screen["<div style='width: 300px; text-align: center; font-family: monospace;'><b>Добро пожаловать в EcoTrack!</b><br/><br/>Android<br/>(или iOS)<br/><br/>Привычек добавлено: 0<br/><br/>┌───────────────────────┐<br/>│  Добавить привычку    │<br/>└───────────────────────┘</div>"]

    style Screen fill:#fafafa,stroke:#333,stroke-width:2px,color:#000
</pre>

<p>Нажал кнопку 3 раза → «Привычек добавлено: 3». Одинаково на обеих платформах.</p>

<hr/>

<h2>5. Подводные камни первого модуля</h2>

<p>Прежде чем перейти к домашнему заданию, давай разберём 3 ошибки, которые я вижу <strong>в каждом втором</strong> KMP-проекте новичка:</p>

<h3>❌ Ошибка 1: Путаница с папками</h3>

<pre>
<code class="language-kotlin">// ❌ Так делать НЕ надо
shared/src/commonMain/kotlin/androidMain/...  ← двойная вложенность, будет конфликтовать</code></pre>

<p><strong>Правильно:</strong> <code>commonMain</code> и <code>androidMain</code> — <strong>сестёрские</strong> папки внутри <code>src/</code>, а не вложенные друг в друга:</p>

<pre>
<code>src/
├── commonMain/    ← уровень 1
├── androidMain/   ← уровень 1 (сестра commonMain)
└── iosMain/       ← уровень 1 (сестра commonMain)</code></pre>

<h3>❌ Ошибка 2: Забыл добавить actual для всех платформ</h3>

<p>Ты объявил <code>expect</code> и написал <code>actual</code> для Android, но забыл для iOS. Компилятор для iOS упадёт:</p>

<pre>
<code>e: Expected an actual declaration for 'getPlatformName(): String'</code></pre>

<p><strong>Решение:</strong> При объявлении <code>expect</code> — сразу создавай файлы <code>actual</code> для <strong>всех</strong> target'ов.</p>

<h3>❌ Ошибка 3: Неправильная версия Compose</h3>

<pre>
<code class="language-kotlin">// ❌ Версии Kotlin и Compose не совпадают
kotlin("multiplatform") version "2.0.20"
kotlin("plugin.compose") version "1.9.22"  // ← несовместимо!</code></pre>

<p><strong>Решение:</strong> Wizard подставит совместимые версии. Если меняешь одну — меняй обе.</p>

<hr/>

<h2>6. Домашнее задание</h2>

<blockquote>
<p>🎯 <strong>Цель:</strong> убедить себя (и компилятор), что ты понял разницу между <code>commonMain</code> и платформенными source set'ами.</p>
</blockquote>

<hr/>

<h3>Задание 1: Измени UI</h3>

<p><strong>Что сделать:</strong> Сделай кнопку зелёной.</p>

<p><strong>Как думать:</strong> В Material 3 цвет кнопки задаётся через <code>ButtonDefaults.buttonColors()</code>. Нам нужно передать <code>containerColor</code>. Импортируй <code>Color</code> из <code>androidx.compose.ui.graphics</code>.</p>

<details><summary>✅ Развернуть: решение</summary>

<pre>
<code class="language-kotlin">import androidx.compose.material3.ButtonDefaults
import androidx.compose.ui.graphics.Color

Button(
    onClick = { count++ },
    colors = ButtonDefaults.buttonColors(
        containerColor = Color(0xFF4CAF50)
    )
) {
    Text("Добавить привычку")
}</code></pre>

</details>

<hr/>

<h3>Задание 2: Expect/Actual с приветствием</h3>

<p><strong>Что сделать:</strong> Создай свой <code>expect/actual</code> — функцию <code>getGreeting(): String</code>.</p>

<p><strong>Как думать:</strong></p>

<ol>
    <li>В <code>commonMain/platform/</code> создай файл <code>Greeting.kt</code> — напиши <code>expect fun getGreeting(): String</code></li>
    <li>В <code>androidMain/platform/</code> — <code>actual fun getGreeting() = "Привет, Android!"</code></li>
    <li>В <code>iosMain/platform/</code> — <code>actual fun getGreeting() = "Привет, iOS!"</code></li>
    <li>В <code>MainScreen</code> добавь <code>Text(getGreeting())</code> под заголовком</li>
</ol>

<p>Подсказка: пакеты должны совпадать с <code>expect</code> (тот же <code>package com.ecotrack.platform</code>).</p>

<details><summary>✅ Развернуть: решение</summary>

<p><strong><code>commonMain/platform/Greeting.kt</code>:</strong></p>

<pre>
<code class="language-kotlin">package com.ecotrack.platform

expect fun getGreeting(): String</code></pre>

<p><strong><code>androidMain/platform/Greeting.kt</code>:</strong></p>

<pre>
<code class="language-kotlin">package com.ecotrack.platform

actual fun getGreeting(): String = "Привет, Android!"</code></pre>

<p><strong><code>iosMain/platform/Greeting.kt</code>:</strong></p>

<pre>
<code class="language-kotlin">package com.ecotrack.platform

actual fun getGreeting(): String = "Привет, iOS!"</code></pre>

<p><strong>В <code>MainScreen</code>:</strong></p>

<pre>
<code class="language-kotlin">// Добавь после заголовка:
Spacer(modifier = Modifier.height(8.dp))
Text(
    text = getGreeting(),
    style = MaterialTheme.typography.titleMedium
)</code></pre>

</details>

<hr/>

<h3>Задание 3: Тёмная тема</h3>

<p><strong>Что сделать:</strong> Добавь переключатель тёмной/светлой темы.</p>

<p><strong>Как думать:</strong></p>

<ul>
    <li>Нужна переменная состояния: <code>var isDarkMode by mutableStateOf(false)</code></li>
    <li>Кнопка (или переключатель) меняет её</li>
    <li>Фон экрана должен меняться: <code>Surface(color = if (isDarkMode) Color(0xFF1A1A2E) else Color.White)</code></li>
</ul>

<p>Подсказка: оберни всё содержимое в <code>Surface</code> с динамическим <code>color</code>.</p>

<details><summary>✅ Развернуть: решение</summary>

<pre>
<code class="language-kotlin">@Composable
fun MainScreen() {
    var count by mutableIntStateOf(0)
    var isDarkMode by mutableStateOf(false)

    val backgroundColor = if (isDarkMode) Color(0xFF1A1A2E) else Color.White
    val textColor = if (isDarkMode) Color.White else Color.Black

    Surface(
        modifier = Modifier.fillMaxSize(),
        color = backgroundColor
    ) {
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(24.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            Text(
                text = "Добро пожаловать в EcoTrack!",
                style = MaterialTheme.typography.headlineMedium,
                color = textColor
            )

            Spacer(modifier = Modifier.height(16.dp))

            Text(
                text = getPlatformName(),
                style = MaterialTheme.typography.bodyMedium,
                color = textColor
            )

            Spacer(modifier = Modifier.height(16.dp))

            Text(
                text = "Привычек добавлено: $count",
                color = textColor
            )

            Spacer(modifier = Modifier.height(16.dp))

            Text(
                text = getGreeting(),
                style = MaterialTheme.typography.titleMedium,
                color = textColor
            )

            Spacer(modifier = Modifier.height(24.dp))

            Button(
                onClick = { count++ },
                colors = ButtonDefaults.buttonColors(containerColor = Color(0xFF4CAF50))
            ) {
                Text("Добавить привычку")
            }

            Spacer(modifier = Modifier.height(12.dp))

            Button(
                onClick = { isDarkMode = !isDarkMode },
                colors = ButtonDefaults.buttonColors(
                    containerColor = if (isDarkMode) Color(0xFFFFC107) else Color(0xFF2196F3)
                )
            ) {
                Text(if (isDarkMode) "☀️ Светлая тема" else "🌙 Тёмная тема")
            }
        }
    }
}</code></pre>

</details>

<hr/>

<h3>Задание 4 (бонус): Счётчик с анимацией</h3>

<p><strong>Что сделать:</strong> При нажатии кнопки цифра «перескакивает» с fade-эффектом.</p>

<p><strong>Как думать:</strong></p>

<ul>
    <li>Используй <code>AnimatedContent</code> из <code>androidx.compose.animation</code></li>
    <li>Ключ анимации — значение <code>count</code></li>
    <li>Вход: <code>fadeIn + slideIn</code></li>
    <li>Выход: <code>fadeOut + slideOut</code></li>
</ul>

<details><summary>✅ Развернуть: решение</summary>

<pre>
<code class="language-kotlin">import androidx.compose.animation.AnimatedContent
import androidx.compose.animation.fadeIn
import androidx.compose.animation.fadeOut
import androidx.compose.animation.slideInHorizontally
import androidx.compose.animation.slideOutHorizontally
import androidx.compose.foundation.layout.size
import androidx.compose.ui.text.style.TextAlign

AnimatedContent(
    targetState = count,
    transitionSpec = {
        (fadeIn(animationSpec = tween(300)) + slideInHorizontally { it / 2 }) to
        (fadeOut(animationSpec = tween(300)) + slideOutHorizontally { -it / 2 })
    }
) { currentCount ->
    Text(
        text = "Привычек добавлено: $currentCount",
        style = MaterialTheme.typography.titleLarge,
        textAlign = TextAlign.Center
    )
}</code></pre>

<blockquote>
<p>⚠️ <code>AnimatedContent</code> — из <code>androidx.compose.animation</code>. В <code>build.gradle.kts</code> commonMain добавь:</p>

<pre>
<code class="language-kotlin">implementation("org.jetbrains.compose.animation:animation:1.6.2")</code></pre>
</blockquote>

</details>

<hr/>

<h3>Критерии сдачи</h3>

<table>
    <thead>
        <tr>
            <th>Задание</th>
            <th>Критерий</th>
            <th>Статус</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>1</td>
            <td>Кнопка зелёная на обеих платформах</td>
            <td>☐</td>
        </tr>
        <tr>
            <td>2</td>
            <td>При запуске на Android видно "Привет, Android!", на iOS — "Привет, iOS!"</td>
            <td>☐</td>
        </tr>
        <tr>
            <td>3</td>
            <td>Кнопка переключает фон экрана, текст остаётся читаемым</td>
            <td>☐</td>
        </tr>
        <tr>
            <td>4</td>
            <td>(бонус) Цифра анимируется при изменении</td>
            <td>☐</td>
        </tr>
    </tbody>
</table>

<hr/>

<h2>Чек-лист перед переходом к Модулю 2</h2>

<ul>
    <li>☐ Понимаю, <strong>зачем</strong> нужен <code>commonMain</code> и <strong>что</strong> в него нельзя класть</li>
    <li>☐ Могу объяснить коллегам, как работает <code>expect/actual</code>, без заиканий</li>
    <li>☐ Мой проект запускается на Android (и на iOS, если есть Mac)</li>
    <li>☐ DOM задание сдано (минимум задания 1–3)</li>
    <li>☐ Я не боюсь открывать <code>build.gradle.kts</code> и что-то в нём менять</li>
</ul>

<hr/>

<blockquote>
<p>📝 <strong>Самопроверка:</strong> Квиз к Модулю 1 — 12 вопросов для проверки понимания. Пройди перед переходом к следующему модулю.</p>
<p>🔜 <strong>Следующий модуль:</strong> Основы Compose Multiplatform. Будет про layout-систему, состояние и первые компоненты Material 3. Если этот модуль для тебя был «лёгким прогоном» — отлично, в следующих будет жарко.</p>
</blockquote>

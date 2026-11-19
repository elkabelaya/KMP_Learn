# 📝 Самопроверка: Модуль 15 — CI/CD & DevOps для KMP

> **Как пользоваться:** отвечай мысленно (или в заметках), а потом раскрывай `<details>` и сверяйся. Не подглядывай до ответа — так тренируется именно понимание, а не узнавание.
>
> **Целевой результат:** 12+ из 15 без подглядывания.

---

## Вопрос 1. Какие события запускают `KMP CI Pipeline` из `.github/workflows/ci.yml`?

- **A)** Только `push` в `main`
- **B)** `push` в `main` и `develop`, а также `pull_request` в `main`
- **C)** `pull_request` во все ветки и `release: created`
- **D)** Только ручной запуск через `workflow_dispatch`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В файле прямо указано:

```yaml
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
```

То есть мы пушим в `main` или `develop` — идёт сборка; открываем PR **в `main`** — тоже идёт сборка. Остальные события workflow не слушает.

**Почему другие не подходят:**
- A — забыли про `develop` и про PR
- C — PR слушается только для `main`, а `release: created` — это триггер **другого** workflow (`release.yml`)
- D — `workflow_dispatch` в этом пайплайне не объявлен

</details>

---

## Вопрос 2. Что делает строка `GRADLE_OPTS: "-Dorg.gradle.daemon=false -Dorg.gradle.parallel=true"` в `env` workflow?

- **A)** Отключает Gradle daemon и включает параллельное выполнение задач
- **B)** Включает Gradle daemon и отключает параллелизм
- **C)** Отключает логирование Gradle и снижает уровень heap
- **D)** Заставляет Gradle собирать модули строго по одному

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Разбираем флаги буквально:
- `-Dorg.gradle.daemon=false` — демон **выключен**
- `-Dorg.gradle.parallel=true` — параллельная сборка **включена**

Это глобальные `env` для всех jobs, поэтому настройка применяется и к lint, и к тестам, и к сборкам.

**Почему другие не подходят:**
- B — прямо наоборот: демон выключен, параллелизм включён
- C — pro guard тут ни при чём; флаги отвечают за daemon и parallel, а не за логи/память
- D — `parallel=true` даёт ровно противоположный эффект

</details>

---

## Вопрос 3. Внимание, ловушка. В `quality-gate` написано `needs: [lint, unit-tests, android-build, ios-build]`. Какая работа **НЕ** является обязательным prerequisites для quality gate, хотя она есть в workflow?

- **A)** `lint`
- **B)** `unit-tests`
- **C)** `coverage`
- **D)** `ios-build`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: C** (`coverage`)

Это вопрос-ловушка: `coverage` действительно есть в файле, но в `needs` у `quality-gate` его **нет**:

```yaml
quality-gate:
  needs: [lint, unit-tests, android-build, ios-build]
```

`coverage` зависит только от `unit-tests` (`needs: unit-tests`) и работает параллельно с quality gate. Значит, падение загрузки отчёта в Codecov **не заблокирует** зелёный статус `ci-success`.

**Почему другие не подходят:**
- A, B, D — все три прямо перечислены в `needs`, это реальные prerequisites
- ⚠️ Практический вывод: если ты хочешь, чтобы coverage был обязательным, его надо добавить в `needs` явно

</details>

---

## Вопрос 4. Зачем в ключе кэша используется `hashFiles('**/*.gradle*', '**/gradle-wrapper.properties')`?

```yaml
key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
```

- **A)** Хэширует содержимое Gradle-файлов: при изменении конфигурации/зависимостей ключ меняется и кэш инвалидируется
- **B)** Генерирует случайный ключ при каждом запуске, чтобы кэш никогда не переиспользовался
- **C)** Кэширует только `gradle-wrapper.properties`, игнорируя остальные файлы
- **D)** Проверяет версию JDK и подставляет её в ключ

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`hashFiles` считает хэш по всем файлам, подходящим под маски. Пока `*.gradle*` и wrapper не менялись — ключ стабилен и кэш переиспользуется. Как только ты добавил зависимость или обновил Gradle — хэш изменился, создаётся новый кэш, а старый остаётся в `restore-keys` как запасной (`${{ runner.os }}-gradle-`).

**Почему другие не подходят:**
- B — случайности здесь нет, хэш детерминирован содержимым файлов
- C — маска `**/*.gradle*` покрывает все build-файлы, а не только wrapper
- D — версия JDK задаётся отдельно через `actions/setup-java`, в ключе её нет

</details>

---

## Вопрос 5. Что означает `if: always()` у шага «Upload lint report»?

- **A)** Отчёт будет загружен даже если предыдущий шаг `./gradlew lint` упал
- **B)** Lint будет запущен даже если весь job отменён пользователем
- **C)** Шаг будет повторяться до бесконечности при ошибке
- **D)** Отчёт загружается только при успешном lint

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

По умолчанию GitHub Actions пропускает шаги после первого провала. `if: always()` снимает это ограничение и гарантирует выполнение шага независимо от результата предыдущих — поэтому артефакт с отчётом всегда можно скачать и разобрать, почему lint упал. Так же помечены «Upload test results» и «Upload coverage report».

**Почему другие не подходят:**
- B — `always()` управляет конкретным шагом, а не перезапуском job'а
- C — это не retry, повторов не будет
- D — как раз наоборот: без `always()` отчёт грузился бы только при успехе

</details>

---

## Вопрос 6. Ты скопировал `ios-build` job и случайно поставил `runs-on: ubuntu-latest`. Что произойдёт?

- **A)** Job упадёт: `xcodebuild` и iOS-симулятор доступны только на macOS, поэтому нужен `macos-latest`
- **B)** Всё соберётся, просто медленнее — Kotlin/Native сам сгенерирует iOS-бинарник
- **C)** Gradle соберёт iOS-приложение без Xcode, потому что используется KMP
- **D)** GitHub Actions автоматически подменит раннер на macOS

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В content.md `ios-build` и `ios-release` используют `runs-on: macos-latest` — и это не случайность. Шаги вызывают `xcodebuild` с `-sdk iphonesimulator` / `-sdk iphoneos`, а Xcode есть только на macOS. KMP здесь не поможет: Kotlin/Native создаёт фреймворк (`:shared:iosSimulatorArm64Binaries`), но сборку и подпись приложения делает Xcode.

**Почему другие не подходят:**
- B — Ubuntu не получится «просто медленнее»: там нет Xcode
- C — KMP не заменяет Xcode для сборки `.app`/`.ipa`
- D — платформа раннера не подменяется автоматически

</details>

---

## Вопрос 7. Откуда в CI берётся keystore для Android-сборки?

- **A)** Декодируется из `secrets.ANDROID_KEYSTORE_BASE64` командой `base64 -d` в `android-app/keystore.jks`
- **B)** Лежит в репозитории в открытом виде — так проще воспроизводить сборку
- **C)** Генерируется Gradle при каждом запуске `assembleDebug`
- **D)** Скачивается с Google Play через сервисный аккаунт

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

В `android-build` и в `android-release` есть шаг:

```bash
echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 -d > android-app/keystore.jks
```

Секрет хранится в виде base64 в настройках репозитория, а в раннере разворачивается во временный файл. Пароли (`ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`) тоже приходят из secrets и попадают в `keystore.properties`.

**Почему другие не подходят:**
- B — хранить keystore в git — прямая угроза безопасности; в модуле специально используется secrets
- C — Gradle не генерирует keystore сам
- D — сервисный аккаунт нужен для загрузки в Play (`deploy-to-stores`), а не для подписи

</details>

---

## Вопрос 8. Зачем в `build-cache.yml` передаётся `cache-encryption-key: ${{ secrets.GRADLE_CACHE_ENCRYPTION_KEY }}`?

- **A)** Чтобы publications/build cache шифровался, а ключ не лежал в исходниках
- **B)** Чтобы зашифровать исходный код репозитория в git
- **C)** Чтобы заменить JDK 17 на более новую версию
- **D)** Чтобы подписать APK перед загрузкой в release

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаг использует `gradle/gradle-build-action@v2` — он публикует и читает Gradle build cache. `cache-encryption-key` шифрует содержимое кэша, а ключ хранится в `secrets`, то есть не попадает в код. Это стандартная практика: кэш может содержать артефакты сборки, и его не хочется светить в открытом виде.

**Почему другие не подходят:**
- B — шифрование git-репозитория тут не происходит
- C — JDK ставится отдельным шагом `actions/setup-java`
- D — подпись APK — это `signRelease` и keystore, а не cache key

</details>

---

## Вопрос 9. Фрагмент `scripts/bump-version.sh`. Текущая версия в `shared/build.gradle.kts` — `2.3.4`. Что получится после `./bump-version.sh minor`?

```bash
VERSION_TYPE=${1:-patch}
CURRENT_VERSION=$(grep "version=" shared/build.gradle.kts | cut -d'"' -f2)
IFS='.' read -r MAJOR MINOR PATCH <<< "$CURRENT_VERSION"

case $VERSION_TYPE in
  minor)
    MINOR=$((MINOR + 1))
    PATCH=0
    ;;
esac

NEW_VERSION="$MAJOR.$MINOR.$PATCH"
```

- **A)** `2.4.0` — увеличился minor, patch сброшен в ноль
- **B)** `2.3.5` — увеличился patch
- **C)** `3.0.0` — увеличился major, остальные сброшены
- **D)** `2.4.4` — увеличился minor, patch остался прежним

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Логика `case` из скрипта:
- `major` → `MAJOR+1`, `MINOR=0`, `PATCH=0`
- `minor` → `MINOR+1`, `PATCH=0` (MAJOR не трогаем)
- `patch` (значение по умолчанию) → `PATCH+1`

Для `2.3.4` и `minor`: MAJOR=2, MINOR=3→4, PATCH=4→0. Итого `2.4.0`.

**Почему другие не подходят:**
- B — это результат вызова `patch` (или вызова без аргумента)
- C — это результат `major`
- D — забыли про `PATCH=0` в ветке `minor`

</details>

---

## Вопрос 10. Ещё ловушка. В `bump-version.sh` комментарии говорят «Update version in gradle.properties». Какой файл **реально** редактируется для версии shared-модуля?

```bash
# Read current version from gradle.properties
CURRENT_VERSION=$(grep "version=" shared/build.gradle.kts | cut -d'"' -f2)

# Update version in gradle.properties
sed -i "s/version=\"[^\"]*\"/version=\"$NEW_VERSION\"/" shared/build.gradle.kts
```

- **A)** `gradle.properties`
- **B)** `shared/build.gradle.kts`
- **C)** `apps/android-app/build.gradle.kts`
- **D)** `ios-app/SkillSync/Info.plist`

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

Вопрос-ловушка на внимательность: комментарии **устарели/не соответствуют коду**. И чтение (`grep`), и запись (`sed -i`) работают с `shared/build.gradle.kts`. `gradle.properties` в скрипте не упоминается в командной части вообще.

Для сравнения, остальные файлы скрипт тоже трогает, но по другим причинам:
- `apps/android-app/build.gradle.kts` → меняет `versionName`
- `ios-app/SkillSync/Info.plist` → меняет `CFBundleShortVersionString` через `PlistBuddy`

**Почему другие не подходят:**
- A — это ловушка: ориентироваться надо на код, а не на комментарий
- C — там правится `versionName`, а не версия shared-модуля
- D — там правится только iOS-версия (`CFBundleShortVersionString`)

</details>

---

## Вопрос 11. Что делает `set -e` в начале `bump-version.sh`?

- **A)** Немедленно завершает скрипт, если какая-то команда вернула ненулевой код
- **B)** Выводит каждую команду перед выполнением
- **C)** Разрешает использование неопределённых переменных без ошибки
- **D)** Отключает весь вывод скрипта в консоль

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`set -e` («errexit») — защита от частично применённого изменения версии. Если, например, `grep` не нашёл `version=` или `sed` не сработал, скрипт остановится, и до `git commit`/`git tag` дело не дойдёт.

**Почему другие не подходят:**
- B — это `set -x`
- C — это про `set -u`; здесь он как раз не включён
- D — `set -e` не влияет на вывод (и в скрипте есть `echo` с прогрессом)

</details>

---

## Вопрос 12. Что запускает `Release Pipeline` (`release.yml`)?

- **A)** `push` в `main`
- **B)** Создание release (`release: types: [created]`)
- **C)** Слияние любого pull request
- **D)** Ежедневный cron по расписанию

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: B**

В `release.yml`:

```yaml
on:
  release:
    types: [created]
```

То есть сначала `bump-version.sh` создаёт тег `vX.Y.Z`, затем в GitHub оформляется release — и только после этого стартуют `android-release`, `ios-release` и `deploy-to-stores`. Это отделяет обычный CI от публикации.

**Почему другие не подходят:**
- A — `push` — триггер `ci.yml` и `build-cache.yml`
- C — PR — триггер `ci.yml`
- D — schedule в модуле не используется

</details>

---

## Вопрос 13. Куда `deploy-to-stores` отправляет Android и iOS-сборки?

- **A)** Android — в трек `internal` Google Play, iOS — в TestFlight через App Store Connect
- **B)** Обе сборки в `production` без промежуточного тестирования
- **C)** Android — в `alpha`, iOS — в `beta`
- **D)** Только в GitHub Releases как вложения

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`deploy-to-stores` выполняется после `needs: [android-release, ios-release]` и содержит два шага:
- `r0adkll/upload-google-play@v1` с `releaseFiles: .../*.aab` и `track: internal`
- `apple-actions/upload-to-app-store-connect@v1` с `build_path: build/ios/ipa/*.ipa`

То есть сначала внутренний трек / TestFlight, а не сразу production. Загрузка в GitHub Releases делается **другим** шагом (`svenstaro/upload-release-action@v2`) внутри jobs сборки.

**Почему другие не подходят:**
- B — модуль явно использует `track: internal` и TestFlight
- C — таких треков в конфиге нет
- D — это делают `android-release`/`ios-release`, а не `deploy-to-stores`

</details>

---

## Вопрос 14. Фрагмент мониторинга. Где находятся реальные реализации `createCrashReporter()`?

```kotlin
// shared/src/commonMain/.../CrashReporter.kt
interface CrashReporter {
    fun logError(error: Throwable, context: Map<String, String> = emptyMap())
    fun logEvent(eventName: String, properties: Map<String, Any> = emptyMap())
    fun setUser(userId: String, email: String? = null, username: String? = null)
}

expect fun createCrashReporter(): CrashReporter
```

- **A)** `AndroidCrashReporter` (Firebase Crashlytics) и `IosCrashReporter` (Firebase Crashlytics) — в платформенных source set'ах через `actual`
- **B)** Прямо в `commonMain` рядом с интерфейсом — `expect` сам себя реализует
- **C)** В `build.gradle.kts` как dependency
- **D)** На сервере: common-код делает HTTP-запрос к Crashlytics

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

Шаблон тот же, что и в базовых модулях: `commonMain` объявляет контракт (`interface CrashReporter` + `expect fun createCrashReporter()`), а каждая платформа даёт `actual`-реализацию. В content.md показаны `AndroidCrashReporter` (`FirebaseCrashlytics.getInstance()...`, `setCustomKey`) и `IosCrashReporter` (`FirebaseCrashlytics.crashlytics()...`, `setCustomValue`). Бизнес-код вызывает `logError`/`logEvent` через интерфейс и не знает о платформе.

**Почему другие не подходят:**
- B — `expect` — это объявление, без `actual` сборка соответствующего таргета упадёт
- C — Gradle-зависимость подключает SDK, но не реализует наш интерфейс
- D — Firebase Crashlytics работает через нативный SDK, а не через HTTP из common-кода

</details>

---

## Вопрос 15. В шаге загрузки coverage есть `fail_ci_if_error: false`. Что это значит?

```yaml
- name: Upload to Codecov
  uses: codecov/codecov-action@v3
  with:
    files: build/reports/jacoco/codeCoverage.xml
    fail_ci_if_error: false
```

- **A)** Ошибка загрузки отчёта в Codecov не роняет CI
- **B)** CI упадёт при любом снижении покрытия
- **C)** Отчёт вообще не будет загружаться
- **D)** Юнит-тесты не будут запускаться

<details>
<summary>💡 Показать ответ и объяснение</summary>

**Правильный ответ: A**

`fail_ci_if_error` управляет поведением **только самого action'а загрузки**: при `false` сетевые проблемы/ошибка Codecov не приводят к провалу job'а. Покрытие при этом всё равно генерируется (`./gradlew createJacocoReport`) и загружается; контроль качества тестов остаётся на `unit-tests`.

**Почему другие не подходят:**
- B — порогов покрытия в этом workflow нет, флаг про ошибки загрузки, а не про метрики
- C — отчёт загружается, флаг лишь смягчает ошибки
- D — тесты — отдельная работа `unit-tests` с `needs` для coverage

</details>

---

## Итог

| Результат | Что делать |
|-----------|-----------|
| 13–15 | 🎉 Отлично! CI/CD пайплайн для KMP — твой следующий шаг — автоматизировать всё остальное |
| 10–12 | 📖 Хорошо. Перечитай секции про `needs`, кэширование и release pipeline |
| 7–9 | ⚠️ Пройди модуль ещё раз: сконцентрируйся на триггерах workflow, runners и secrets |
| < 7 | 🆘 Вернись к `.github/workflows/*.yml` и разбери каждый шаг по строкам, потом пройди квиз снова |

---

> 🔜 Квиз к Модулю 16 (Mentorship & Professional Growth) — в соседней папке `Senior_Track/Module_16_Mentorship_Processes`.

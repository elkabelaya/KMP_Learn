# Prompt'ы для Documentation и Production

## 📝 API Documentation

### Prompt: Генерация KDoc documentation

```
Ты - Technical Writer с опытом в Kotlin API documentation.

Создай comprehensive KDoc комментарии для следующего кода:

[ВСТАВИТЬ КОД - класс, функцию или модуль]

Требования к KDoc:
1. **Summary:** Краткое описание (1-2 предложения) что делает класс/функция
2. **Description:** Подробное описание поведения, edge cases  
3. **Parameters:** Для каждого параметра - название, тип, описание, constraints
4. **Returns:** Что возвращает и в каких случаях  
5. **Throws:** Какие exceptions может бросить (если не использует Result)
6. **Example:** Минимальный рабочий пример использования  
7. **See Also:** Ссылки на связанные классы/функции

Формат KDoc:
```kotlin
/**
 * [Summary sentence.](#summary)
 * 
 * [Detailed description paragraph(s). Explain:
 * - What this does  
 * - When to use it
 * - Important behavior or edge cases
 * - Thread safety considerations (если применимо)
 * ](#description)
 * 
 * @param param1 [Description of parameter. Include:
 * - What it represents  
 * - Valid values or constraints
 * - Default behavior if not provided
 * ](#param1)
 * 
 * @return [Description of return value. Include:
 * - What the return represents  
 * - Possible values or states
 * - When null или error может быть возвращено
 * ](#return)
 * 
 * @throws [ExceptionType] [When this exception is thrown](#throws)
 * 
 * @see [RelatedClass] [Brief description of relationship](#seealso)
 * 
 * @sample
 * ```kotlin
 * // Minimal working example  
 * val result = myFunction(param1 = "value")
 * ```
 */
[CODE]
```

Создай KDoc для всех public классов, функций и properties.
Будь concise но comprehensive - developers должны понять API без чтения source code.
```

---

### Prompt: Создание README.md для модуля/библиотеки

```
Ты - Senior Developer и Technical Writer.

Создай профессиональный README.md для [MODULE/LIBRARY NAME].

Контекст:
- Тип: [KMP library / Feature module / Utility package]  
- Назначение: [Описание что делает модуль]
- Целевая аудитория: [KMP developers / Mobile developers / etc.]

Структура README:

```markdown
# [Module Name] 🚀

[Optional badge: version, license, platforms support]

## 📖 Overview
[2-3 paragraph description:  
- What problem this solves
- Key capabilities  
- How it fits in larger architecture]

## ✨ Features
- [Feature 1 with brief description]  
- [Feature 2 with brief description]
- [Feature 3 with brief description]

## 📦 Installation

### Dependencies
```kotlin
// build.gradle.kts для consumers
dependencies {
    implementation("com.example:[module-name]:[version]")
}
```

### Platform Requirements  
- Kotlin: [version]+
- Android: minSdk [version]  
- iOS: deploymentTarget [version]
- Desktop: [requirements если применимо]

## 🚀 Quick Start

```kotlin
// Minimal working example - copy-paste ready
import com.example.[module].*

class MyFeature {
    private val myService = MyService()
    
    suspend fun doSomething() {
        val result = myService.execute(input)
        result.fold(
            onSuccess = { /* handle success */ },  
            onError = { /* handle error */ }
        )
    }
}
```

## 📚 API Reference

### [Main Class/Interface 1]
**Purpose:** [What it does]

```kotlin
// Key methods with brief descriptions
interface MyService {
    suspend fun execute(input: Input): Result<Output>
    // ...
}
```

[Link to full API docs if available]

### [Main Class/Interface 2]
...

## 🔧 Configuration  
[Если есть configuration options:
- What can be configured
- Default values  
- How to customize]

## 📖 Usage Examples

### Example 1: [Common Use Case]
```kotlin
// Complete working example
```

### Example 2: [Advanced Use Case]  
```kotlin
// More complex scenario
```

## 🧪 Testing
[How to test code that uses this module:
- Test dependencies needed  
- Mock setup examples
- Common test patterns]

## 🐛 Troubleshooting

### [Common Issue 1]
**Symptom:** [What user sees]  
**Cause:** [Why it happens]  
**Solution:** [How to fix]

### [Common Issue 2]
...

## 📝 Best Practices  
- [Best practice 1 with explanation]
- [Best practice 2 with explanation]

## ⚠️ Known Limitations  
- [Limitation 1 and workaround if any]
- [Limitation 2]

## 🤝 Contributing  
[How others can contribute:
- Reporting bugs
- Suggesting features  
- Submitting PRs]

## 📄 License  
[License type - MIT, Apache 2.0, etc.]

## 🔗 Related
- [Link to related modules]  
- [Link to main project README]
```

Создай comprehensive README в этом формате.
Используй emojis для visual organization но не переусердствуй.
```

---

## 🏭 Production Readiness

### Prompt: Создание Production Checklist

```
Ты - Senior DevOps Engineer с опытом в mobile application deployment.

Создай comprehensive production readiness checklist для KMP приложения [APP NAME].

Категории проверки:

## 1. Code Quality ✅
- [ ] All compiler warnings resolved  
- [ ] No TODO comments или temporary code
- [ ] Code coverage ≥80% (unit tests)  
- [ ] All integration tests passing
- [ ] No debug.log() или println() statements

## 2. Security 🔒  
- [ ] All API keys и secrets в environment variables (не hardcoded)
- [ ] TLS certificate pinning configured и tested  
- [ ] Database encryption enabled
- [ ] Secure storage для sensitive data (tokens, credentials)
- [ ] ProGuard/R8 obfuscation enabled (Android)  
- [ ] Bitcode и stripping enabled (iOS)
- [ ] No debug symbols в production builds

## 3. Performance ⚡  
- [ ] App cold start time < [TARGET] seconds
- [ ] Memory usage < [TARGET] MB на typical screens  
- [ ] No memory leaks (verified с LeakCanary или Instruments)
- [ ] Network requests optimized (caching, batching)  
- [ ] Image loading optimized (compression, lazy loading)

## 4. Error Handling 🛡️  
- [ ] All network errors handled gracefully
- [ ] User-friendly error messages (no technical details)  
- [ ] Crash reporting configured (Sentry, Firebase Crashlytics)
- [ ] Error analytics enabled  
- [ ] Retry logic для transient failures

## 5. Analytics & Monitoring 📊
- [ ] Analytics SDK integrated (Firebase, Mixpanel, etc.)  
- [ ] Key user events tracked (onboarding, conversions)
- [ ] Performance monitoring enabled  
- [ ] Custom dashboards created для key metrics

## 6. Platform-Specific ✅

### Android:
- [ ] ProGuard/R8 rules configured  
- [ ] App signing with production keystore
- [ ] Play Console app bundle uploaded и tested  
- [ ] Content rating submitted
- [ ] Privacy policy URL configured

### iOS:  
- [ ] App Store Connect metadata prepared
- [ ] Production provisioning profile configured  
- [ ] App icon и screenshots в required sizes
- [ ] Privacy nutrition labels completed  
- [ ] In-app purchase configuration (если применимо)

## 7. Legal & Compliance ⚖️  
- [ ] Privacy policy published и linked в app
- [ ] Terms of service available  
- [ ] GDPR compliance (data deletion, export)
- [ ] Age restrictions configured (если применимо)  
- [ ] Third-party library licenses documented

## 8. Rollout Strategy 🚀
- [ ] Staged rollout plan (1% → 5% → 25% → 100%)  
- [ ] Rollback procedure documented и tested
- [ ] Feature flags configured для risky changes  
- [ ] Monitoring alerts configured для error rates

## 9. Documentation 📚
- [ ] README.md updated с version и changelog  
- [ ] API documentation current (если есть public API)
- [ ] Internal runbook для operations team  
- [ ] User guide или help documentation

## 10. Post-Launch 📈
- [ ] Monitoring dashboard accessible  
- [ ] On-call rotation scheduled
- [ ] First 24h monitoring plan  
- [ ] User feedback collection mechanism

Для каждого пункта укажи:
- **Status:** [ ] Not Started / [🟢] Complete / [🟡] In Progress
- **Owner:** [Team member responsible]  
- **Notes:** [Any additional context или blockers]

Создай checklist в формате таблицы с колонками:
| Category | Item | Status | Owner | Notes | Priority |

Приоритеты: 🔴 Critical (must have) / 🟡 High (should have) / 🟢 Medium (nice to have)
```

---

### Prompt: Создание Release Notes

```
Ты - Product Manager и Technical Writer.

Создай release notes для версии [VERSION] приложения [APP NAME].

Контекст релиза:
- Тип: [Major / Minor / Patch]  
- Дата релиза: [DATE]
- Ключевые изменения: [LIST OF CHANGES]

Структура release notes:

```markdown
# 🎉 [App Name] [Version] - [Release Theme или Tagline]

**Released:** [Date]  
**Download:** [Android link] | [iOS link]

---

## 🚀 What's New

### ✨ New Features
- **[Feature 1 Name]** - [Brief description of what it does и why users will love it]
  - [Key capability 1]  
  - [Key capability 2]
  
- **[Feature 2 Name]** - [Brief description]
  - [Key capability]

### 🎨 Improvements  
- **[Area 1]** - [What improved и benefit to users]
- **[Area 2]** - [What improved]

### 🐛 Bug Fixes  
- Fixed: [Bug description в user-friendly language]
- Fixed: [Another bug fix]

### 🔒 Security  
- [Security improvements или updates - если применимо]

---

## 📱 Platform Notes

### Android
- [Android-specific changes или requirements]  
- Minimum Android version: [VERSION]

### iOS  
- [iOS-specific changes или requirements]
- Minimum iOS version: [VERSION]

---

## 🔧 For Developers  
[Если есть breaking changes или API updates:
- What changed
- Migration guide link  
- Deprecation notices]

---

## 🙏 Credits & Thanks
- Special thanks to [contributors, beta testers, etc.]  
- Built with love using Kotlin Multiplatform ❤️

---

## 📞 Feedback & Support
Have feedback или found a bug? Let us know:  
- [Email link]  
- [GitHub Issues link]
- [Twitter handle]

---

**Upgrade now и enjoy the new features! 🚀**
```

Напиши engaging release notes в этом формате.
Используй user-friendly language (избегай technical jargon для end users).
Фокусируйся на benefits, не только features.
```

---

## 🔄 Migration Guides

### Prompt: Создание Migration Guide между версиями

```
Ты - Senior Developer и Technical Writer.

Создай migration guide от [OLD VERSION] к [NEW VERSION] для [LIBRARY/MODULE NAME].

Breaking changes в этой версии:
[LIST OF BREAKING CHANGES - API removals, behavior changes, etc.]

Структура migration guide:

```markdown
# Migration Guide: [Library] [Old Version] → [New Version] 🔄

**Published:** [Date]  
**Difficulty:** [Easy / Medium / Complex]  
**Estimated Time:** [X hours для типичного проекта]

---

## 📋 Overview
[2-3 paragraphs:  
- What major changes в этой версии
- Why breaking changes были necessary  
- Benefits of upgrading]

---

## ⚠️ Breaking Changes

### 1. [Change Category 1]

**What Changed:**
[Description of what broke]

**Old Code (v[X]):**
```kotlin
// Example of old API usage  
oldApi.doSomething()
```

**New Code (v[Y]):**  
```kotlin
// Updated API usage
newApi.doSomething()
```

**Migration Steps:**
1. [Step 1 - what to change]
2. [Step 2 - any additional setup]  
3. [Step 3 - testing recommendations]

**Impact:** [High / Medium / Low]  
**Affected Code:** [Which parts of typical codebase]

---

### 2. [Change Category 2]
...

---

## 🔧 Automated Migration (если применимо)

### Search & Replace
[Если есть простые replacements:
| Find | Replace | Files |
|------|---------|-------|  
| `oldImport` | `newImport` | All Kotlin files |
]

### Code Migration Script (если есть)
```bash
# Command to run automated migration  
./migrate-to-v[Y].sh
```

---

## 📚 Detailed API Changes

### Deprecated APIs  
[Table of deprecated items с replacements:
| Old API | Status | Replacement | Notes |
|---------|--------|-------------|-------|  
| `oldFunction()` | Deprecated | `newFunction()` | Behavior identical |
]

### Removed APIs  
[Table of removed items с migration path:
| Removed API | Migration Path | Example |
|-------------|----------------|---------|  
| `oldClass` | Use `newClass` instead | See example below |
]

### New Features Worth Adopting  
[Highlight new APIs that improve old patterns:
- [New API 1] - Replaces [old pattern], benefits: [...]  
- [New API 2] - Simplifies [common task]
]

---

## 🧪 Testing Your Migration

### Checklist после migration:
- [ ] Project compiles без warnings  
- [ ] All unit tests passing
- [ ] Integration tests passing  
- [ ] Manual testing of critical workflows
- [ ] Performance benchmarks (если применимо)

### Common Migration Issues:

**Issue:** [Common problem 1]  
**Solution:** [How to fix]

**Issue:** [Common problem 2]  
**Solution:** [How to fix]

---

## 📖 Additional Resources
- [Changelog link]  
- [API Documentation link]
- [Example projects using new version]  
- [Migration FAQ]

---

## 🆘 Need Help?
- [GitHub Discussions link]  
- [Stack Overflow tag: library-name]
- [Email support]

---

**Happy migrating! 🚀**  
[Library Team]
```

Создай comprehensive migration guide в этом формате.
Будь specific с кодом и examples - avoid vague descriptions.
```

---

## 📝 Как использовать эти prompt'ы

### Для Documentation:
- **KDoc:** После написания нового API - сгенерируйте documentation  
- **README:** При создании новой библиотеки или major module
- **Migration Guides:** При breaking changes в public API

### Для Production:  
- **Production Checklist:** Перед каждым major release
- **Release Notes:** При каждом public release (major, minor)  
- **Migration Guides:** При major version bumps с breaking changes

### Best Practices:
- ✅ Generate documentation early - не оставляйте на последний момент  
- ✅ Keep docs close to code - обновляйте при API changes
- ✅ Test migration guides на real projects перед publication  
- ✅ Use versioned docs для different major versions

---

**Well-documented code is loved code! 📚✨**

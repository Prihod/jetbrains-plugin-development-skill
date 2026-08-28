---
title: Create a new plugin project
tags: workflow, build, gradle, scaffolding
verify: PLUGIN_TEMPLATE_HOME="${PLUGIN_TEMPLATE_HOME:?}"; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' references/workflow-create-plugin.md); for p in 'id("org.jetbrains.intellij.platform.settings") version "2.16.0"' 'intellijIdea("2025.2.6.2")' 'testFramework(TestFrameworkType.Platform)' 'kotlin.stdlib.default.dependency = false' 'CHANGELOG.md' 'Version is missing: Unreleased'; do printf '%s\n' "$body" | grep -qF "$p" || exit 1; done && grep -q 'id("org.jetbrains.intellij.platform.settings") version "2.16.0"' "$PLUGIN_TEMPLATE_HOME/settings.gradle.kts" && grep -q 'intellijIdea("2025.2.6.2")' "$PLUGIN_TEMPLATE_HOME/build.gradle.kts" && grep -q 'testFramework(TestFrameworkType.Platform)' "$PLUGIN_TEMPLATE_HOME/build.gradle.kts" && grep -q 'kotlin.stdlib.default.dependency = false' "$PLUGIN_TEMPLATE_HOME/gradle.properties" && grep -q 'id("org.jetbrains.changelog")' "$PLUGIN_TEMPLATE_HOME/build.gradle.kts" && test -f "$PLUGIN_TEMPLATE_HOME/CHANGELOG.md"
---

## Create a new plugin project

One sequence takes an empty directory to a plugin that builds, sandboxes, and passes its first test. Skipping a step is how AP-09/AP-10 sneak in unnoticed.

### Preconditions

- A JDK the target platform version supports, and network access for the first Gradle sync (`SKILL.md`'s Step 0 covers detecting the target version). **The first sync for one target `intellijIdea(...)` version downloads and extracts roughly 6.5 GB** (observed: a 1.7 GB installer artifact under `~/.gradle/caches/modules-2/files-2.1/idea/`, plus a ~4.8 GB extracted/instrumented copy under `~/.gradle/caches/<gradle-version>/transforms/`) — confirm at least that much free disk first; this has exhausted disk and stopped work outright on a constrained machine. `verifyPlugin` has its own, separate multi-GB default — see [compat-verifier-ides-and-cost.md](compat-verifier-ides-and-cost.md).
- `$PLUGIN_TEMPLATE_HOME` available locally as the reference layout — see [source-lookup.md](source-lookup.md) if it is not yet cloned.

### Steps

1. `settings.gradle.kts` — inside `plugins { }`, apply the settings plugin:
   ```kotlin
   id("org.jetbrains.intellij.platform.settings") version "2.16.0"
   ```
2. `build.gradle.kts` — inside `dependencies { intellijPlatform { } }`, declare the platform dependency and the test framework:
   ```kotlin
   intellijIdea("2025.2.6.2")
   testFramework(TestFrameworkType.Platform)
   ```
3. `gradle.properties` — opt out of the bundled Kotlin stdlib (AP-09):
   ```properties
   kotlin.stdlib.default.dependency = false
   ```
4. `CHANGELOG.md` — mandatory the moment `org.jetbrains.changelog` is applied, which the template does and `SKILL.md`'s Baseline table pins. With that plugin applied and no `CHANGELOG.md` in the project root, `patchPluginXml` fails before producing anything — `Version is missing: Unreleased` with the configuration cache on, `Failed to query the value of extension 'pluginConfiguration' property 'changeNotes'` with it off — so the first `./gradlew build` cannot succeed. Reproduced on this Baseline; the task needs only the file to exist, and its `## [Unreleased]` section is what feeds `<change-notes>` into the patched descriptor. Copy `$PLUGIN_TEMPLATE_HOME/CHANGELOG.md`'s shape, or leave the changelog plugin out of `build.gradle.kts` if the project will not keep one:
   ```markdown
   ## [Unreleased]

   ### Added

   - Initial release.
   ```
5. Wrapper — copy `gradle/wrapper/`, `gradlew`, `gradlew.bat` from `$PLUGIN_TEMPLATE_HOME`, or generate one at the Gradle version recorded in `SKILL.md`'s Baseline table with `gradle wrapper --gradle-version <version>`.
6. Layout — `src/main/kotlin`, `src/main/resources/META-INF`, `src/test/kotlin`; see [setup-build.md](setup-build.md)'s Project layout.
7. `plugin.xml` — write the minimal descriptor; see [setup-plugin-xml.md](setup-plugin-xml.md), which cites `$PLUGIN_TEMPLATE_HOME`'s own file element by element.
8. First service — a light `@Service`-annotated class needs no XML registration; see [extend-services.md](extend-services.md). `$PLUGIN_TEMPLATE_HOME/src/main/kotlin/org/jetbrains/plugins/template/services/MyProjectService.kt` is a real, minimal one.
9. First test — a `BasePlatformTestCase` (Light level) subclass is the default; see [testing-levels-fixtures.md](testing-levels-fixtures.md). `$PLUGIN_TEMPLATE_HOME/src/test/kotlin/org/jetbrains/plugins/template/MyPluginTest.kt` is a real one.

### Validation

Never assume Gradle task names. List what this project actually offers first:

```
./gradlew tasks
```

Then run the build task, the test task, and — once a target range is set (see [compat-range-and-verifier.md](compat-range-and-verifier.md)) — the verifier task, using exactly the names `./gradlew tasks` printed for this project, not ones recalled from another plugin or an older Gradle plugin version.

### Common mistakes

AP-09 (missing `kotlin.stdlib.default.dependency = false` — the Kotlin Gradle plugin bundles a second stdlib copy that can conflict with the platform's own at runtime, with no build-time signal) and AP-10 (a leftover 1.x `org.jetbrains.intellij` plugin id instead of the 2.x `org.jetbrains.intellij.platform` family — it still resolves and builds, it just isn't the current baseline) both pass a build with no warning; see [antipatterns-build.md](antipatterns-build.md) for what else a current template deliberately omits.

### References

[setup-build.md](setup-build.md); [setup-plugin-xml.md](setup-plugin-xml.md); [extend-services.md](extend-services.md); [testing-levels-fixtures.md](testing-levels-fixtures.md); [antipatterns-build.md](antipatterns-build.md); [compat-range-and-verifier.md](compat-range-and-verifier.md); [source-lookup.md](source-lookup.md); `SKILL.md`'s Baseline table; `$PLUGIN_TEMPLATE_HOME`.

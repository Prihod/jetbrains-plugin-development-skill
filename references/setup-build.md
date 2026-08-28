---
title: Configuring the 2.x build
tags: build, gradle, sandbox
verify: PLUGIN_TEMPLATE_HOME="${PLUGIN_TEMPLATE_HOME:?}"; f=references/setup-build.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF '2.16.0' || exit 1; printf '%s\n' "$body" | grep -qF '2025.2.6.2' || exit 1; grep -q 'id("org.jetbrains.intellij.platform.settings") version "2.16.0"' "$PLUGIN_TEMPLATE_HOME/settings.gradle.kts" && grep -q 'intellijIdea("2025.2.6.2")' "$PLUGIN_TEMPLATE_HOME/build.gradle.kts"
---

## Configuring the 2.x build

The IntelliJ Platform Gradle Plugin 2.x applies in `settings.gradle.kts`, not
`build.gradle.kts`, and drops `gradle.properties` platform coordinates for a typed
dependency call. See `SKILL.md`'s Baseline table for the exact versions in force.

**Wrong:**

```properties
# gradle.properties — pre-2.x convention; the 2.x plugin does not read these keys
platformType = IU
platformVersion = 2025.2
```

**Right:**

```kotlin
// settings.gradle.kts
plugins {
    id("org.gradle.toolchains.foojay-resolver-convention") version "1.0.0"
    id("org.jetbrains.intellij.platform.settings") version "2.16.0"
}

dependencyResolutionManagement {
    repositories {
        mavenCentral()
        intellijPlatform { defaultRepositories() }
    }
}
```

```kotlin
// build.gradle.kts
dependencies {
    intellijPlatform {
        intellijIdea("2025.2.6.2")
        testFramework(TestFrameworkType.Platform)
    }
}
```

`SKILL.md`'s Baseline table is the authoritative record of these version numbers; the project you are working in wins over both.

What a current `plugin.xml` and build deliberately omit — AP-09, AP-10, and the rest
of the absent-list — is in `antipatterns-build.md`; do not add any of it from memory.

**Project layout** (from `intellij-platform-plugin-template`):

```
src/main/kotlin/…              plugin code
src/main/resources/META-INF/plugin.xml
src/test/kotlin/…              tests
build.gradle.kts, settings.gradle.kts, gradle.properties
```

**Sandbox.** The 2.x plugin runs a sandboxed IDE at `.intellijPlatform/sandbox/`
(under the project root, alongside the dependency cache), not the 1.x-era
`build/idea-sandbox/`. The template's own `.gitignore` lists `.intellijPlatform`
next to `.gradle`, `.idea`, `.kotlin` and `build` — treat all of them as generated.
Do not hand-configure a different sandbox location unless a task requires it.

The Kotlin stdlib trap (AP-09) lives in the same file — `gradle.properties` must
carry `kotlin.stdlib.default.dependency = false`, or the Kotlin Gradle plugin bundles
a second copy of the stdlib that can conflict with the platform's own at runtime.

Reference: `intellij-platform-plugin-template`'s `settings.gradle.kts`,
`build.gradle.kts`, `gradle.properties`, `.gitignore`.

---
title: Build configuration traps — Kotlin stdlib and legacy Gradle
tags: build, gradle, kotlin-stdlib
verify: PLUGIN_TEMPLATE_HOME="${PLUGIN_TEMPLATE_HOME:?}"; f=references/antipatterns-build.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'kotlin.stdlib.default.dependency' || exit 1; grep -n "kotlin.stdlib.default.dependency" "$PLUGIN_TEMPLATE_HOME/gradle.properties"
---

## AP-09: Missing `kotlin.stdlib.default.dependency = false`

Without this flag in `gradle.properties`, the Kotlin Gradle plugin bundles its own
Kotlin stdlib alongside the platform's own copy, and the two versions can conflict
at runtime with no build-time signal. See the Baseline table in `SKILL.md`.

**Wrong:**

```properties
# gradle.properties — flag absent; stdlib bundling defaults on
```

**Right:**

```properties
kotlin.stdlib.default.dependency = false
```

**Caught by:** runtime (stdlib version conflicts; the build itself succeeds)

Reference: `intellij-platform-plugin-template`'s `gradle.properties`; Baseline table in `SKILL.md`.

## AP-10: Legacy Gradle IntelliJ Plugin 1.x configuration in a new project

`id("org.jetbrains.intellij")` (1.x) still resolves and builds; it just isn't what
current templates and platform versions are validated against. See the Baseline
table in `SKILL.md` for the current `org.jetbrains.intellij.platform` setup.

**Wrong:**

```kotlin
// build.gradle.kts
plugins { id("org.jetbrains.intellij") version "1.17.4" } // legacy 1.x plugin id
```

**Right:**

```kotlin
// settings.gradle.kts
plugins { id("org.jetbrains.intellij.platform.settings") version "2.16.0" }
```

**Caught by:** nothing (the old plugin still builds; it just isn't the current baseline)

Reference: Baseline table in `SKILL.md`; `intellij-platform-plugin-template`'s
`build.gradle.kts` and `settings.gradle.kts`.

Also absent from the current template — do not add them from memory: `platformType` /
`platformVersion` in `gradle.properties`, `gradle/libs.versions.toml`, `<idea-version>`
in `plugin.xml`, an `intellijPlatform { pluginConfiguration { } }` block.

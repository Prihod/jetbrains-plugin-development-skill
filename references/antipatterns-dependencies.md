---
title: Undeclared bundled-plugin dependency traps
tags: dependencies, bundled-plugins, plugin-xml
verify: IJ_SAMPLES="${IJ_SAMPLES:?}"; f=references/antipatterns-dependencies.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'bundledPlugin' || exit 1; grep -n "bundledPlugin" "$IJ_SAMPLES/project_model/build.gradle.kts"
---

## AP-12: Using a bundled plugin's API without declaring the dependency

In IntelliJ IDEA the class is already on the classpath (many bundled plugins are
enabled by default), so the code compiles and runs there. Ship it, and it fails to
load in PhpStorm, WebStorm, or any target IDE that doesn't bundle that plugin.

**Wrong:**

```kotlin
// build.gradle.kts has no bundledPlugin(...) for the plugin providing this import
import com.intellij.psi.impl.source.PsiJavaFileImpl
```

**Right:**

```kotlin
// build.gradle.kts
intellijPlatform { bundledPlugin("com.intellij.java") }
```

```xml
<!-- plugin.xml -->
<depends>com.intellij.java</depends>
```

**Caught by:** runtime (in the target IDE; it works fine in the IDE you built against)

Reference: `project_model/build.gradle.kts`, `framework_basics/src/main/resources/META-INF/plugin.xml`
in `intellij-sdk-code-samples`.

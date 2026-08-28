---
title: Where a dependency comes from — platform, bundled, external
tags: dependencies, build, plugin-xml, origin
verify: IJ_SAMPLES="${IJ_SAMPLES:?}"; HOME="${HOME:?}"; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' references/setup-dependency-origin.md); norm=$(printf '%s' "$body" | tr '\n' ' ' | tr -s ' '); JAR=$(find "$HOME/.gradle/caches/modules-2" -name 'intellij-platform-gradle-plugin-*.jar' | head -1); test -n "$JAR" || exit 1; api=$(javap -classpath "$JAR" -p org.jetbrains.intellij.platform.gradle.extensions.IntelliJPlatformDependenciesExtension) || exit 1; for m in local compatiblePlugin bundledPlugin plugin; do printf '%s' "$norm" | grep -qF "\`$m(" || printf '%s' "$norm" | grep -qF "$m(" || exit 1; printf '%s\n' "$api" | grep -q "void $m(java.lang.String)" || exit 1; done; printf '%s\n' "$api" | grep -q 'void local(java.nio.file.Path)' || exit 1; printf '%s\n' "$api" | grep -q 'void compatiblePlugin(org.gradle.api.provider.Provider' || exit 1; printf '%s' "$norm" | grep -qF 'local(String|File|Path|Directory|Provider<?>)' || exit 1; printf '%s' "$norm" | grep -qF 'compatiblePlugin(String|Provider<String>)' || exit 1; grep -n "bundledPlugin" "$IJ_SAMPLES/project_model/build.gradle.kts"
---

## Where a dependency comes from

Every symbol you import falls into exactly one of three origins, and each needs a
different declaration in both the build script and `plugin.xml`. Getting the origin
wrong compiles fine and fails only in the target IDE.

**Wrong:**

```kotlin
// build.gradle.kts — treating a bundled plugin as if it were external
intellijPlatform {
    plugin("com.intellij.java", "1.0")   // com.intellij.java is not on Marketplace
}
```

**Right:**

```kotlin
// build.gradle.kts
intellijPlatform {
    bundledPlugin("com.intellij.java")   // ships inside IntelliJ IDEA / other IDEs
}
```

```xml
<!-- plugin.xml -->
<depends>com.intellij.java</depends>
```

| Origin | Symbol lives in | Build declaration | `plugin.xml` |
|---|---|---|---|
| Platform | `intellijPlatform { intellijIdea(...) }` itself — core modules | already covered, nothing extra | none beyond `com.intellij.modules.platform` |
| Bundled plugin | a plugin shipped inside IDE distributions (`com.intellij.java`, `org.jetbrains.kotlin`, …) | `intellijPlatform { bundledPlugin("<id>") }` (or `bundledModule("<id>")` for a bundled content module) | `<depends>id</depends>` |
| External plugin | a separate plugin from Marketplace, not bundled with any IDE | `intellijPlatform { plugin("<id>", "<version>") }` | `<depends>id</depends>` |

**How to tell which:** locate the class first — `source-lookup.md`'s
[Finding sources](source-lookup.md#finding-sources) and
[Search recipes](source-lookup.md#search-recipes) give the mechanical steps (which
jar or plugin directory a class comes from). If it resolves inside an installed IDE's
own `lib/` or a core platform module, it's platform. If it resolves inside that IDE's
`plugins/<id>/`, it's bundled **in that IDE** — check the target IDE too, since
bundling differs between IntelliJ IDEA, PhpStorm, WebStorm, and others. If it isn't
present in any installed IDE's `plugins/` directory, it's external, and the user's
target IDE must have it installed for your plugin to load.

## `local()` and `compatiblePlugin()` — how the artefact is obtained

The rows above say *what* a symbol is; these two say *how* it is fetched, inside the same block:

```kotlin
local("/path/to/Installed IDE.app")     // platform source: an IDE already on this machine
compatiblePlugin("com.jetbrains.php")   // plugin version resolved to match that platform
```

`local(...)` stands in for `intellijIdea("<version>")` — no download, but you inherit that
install's build number, JBR and Kotlin metadata version, which can be newer than the Baseline
and break `compileKotlin` ([source-lookup-target-build.md](source-lookup-target-build.md)).
`compatiblePlugin(id)` stands in for `plugin(id, version)`. Neither is in
`$PLUGIN_TEMPLATE_HOME` or `./gradlew help`; when template and `help` both miss a block, read
its signature off the resolved Gradle plugin jar instead of guessing:

```bash
JAR=$(find ~/.gradle/caches/modules-2 -name 'intellij-platform-gradle-plugin-*.jar' | head -1)
javap -classpath "$JAR" -p org.jetbrains.intellij.platform.gradle.extensions.IntelliJPlatformDependenciesExtension | grep -E ' (local|compatiblePlugin|plugin|bundledPlugin)\('
```

That prints `local(String|File|Path|Directory|Provider<?>)` and `compatiblePlugin(String|Provider<String>)`
on the **main** dependencies extension — not the verifier's `ides { local(...) }` ([compat-verifier-ides-and-cost.md](compat-verifier-ides-and-cost.md)).

Forgetting the `plugin.xml` half entirely, once the build half is right, is AP-12 —
see `antipatterns-dependencies.md`.

Reference: `project_model/build.gradle.kts` in `intellij-sdk-code-samples`;
`framework_basics/src/main/resources/META-INF/plugin.xml`.

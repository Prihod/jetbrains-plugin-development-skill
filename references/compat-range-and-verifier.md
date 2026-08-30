---
title: Compatibility is a verified range, not a claim
tags: compatibility, since-until, plugin-verifier, bundled-plugins
verify: PLUGIN_TEMPLATE_HOME="${PLUGIN_TEMPLATE_HOME:?}"; HOME="${HOME:?}"; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' references/compat-range-and-verifier.md); norm=$(printf '%s' "$body" | tr '\n' ' ' | tr -s ' '); for s in 'build/tmp/patchPluginXml/plugin.xml' 'pluginConfiguration { ideaVersion' 'ides { local(' 'ides { create(IntelliJPlatformType.IntellijIdeaCommunity, "2023.3") }' 'do not widen' '0. **The build you already compile against**' 'binds it at zero download cost'; do printf '%s' "$norm" | grep -qF "$s" || exit 1; done; rung0=references/compat-verifier-ides-and-cost.md; printf '%s' "$norm" | grep -qF "($(basename $rung0))" || exit 1; grep -qx '## The floor build is already unpacked on disk' "$rung0" || exit 1; grep -qF 'ls -d ~/.gradle/caches' "$rung0" || exit 1; JAR=$(find "$HOME/.gradle/caches/modules-2" -name 'intellij-platform-gradle-plugin-*.jar' | head -1); test -n "$JAR" || exit 1; javap -classpath "$JAR" -p 'org.jetbrains.intellij.platform.gradle.extensions.IntelliJPlatformExtension$PluginVerification$Ides' | grep -q 'void create(java.lang.Object, java.lang.String)' || exit 1; javap -classpath "$JAR" 'org.jetbrains.intellij.platform.gradle.IntelliJPlatformType' | grep -q ' IntellijIdeaCommunity;' || exit 1; test "$(grep -c '<idea-version' "$PLUGIN_TEMPLATE_HOME/src/main/resources/META-INF/plugin.xml")" = 0 && test -f "$PLUGIN_TEMPLATE_HOME/.run/Run Verifications.run.xml" && grep -q "verifyPlugin" "$PLUGIN_TEMPLATE_HOME/.run/Run Verifications.run.xml"
---

## Compatibility is a verified range, not a claim

The version range in `plugin.xml` and whether the plugin actually loads in a given
IDE are two different questions; only running the Plugin Verifier answers the second.

`$IJ_SAMPLES/theme_basics/resources/META-INF/plugin.xml` writes the range by hand:
`<idea-version since-build="221"/>`. The plugin template ships **without** an
`<idea-version>` element at all
(`$PLUGIN_TEMPLATE_HOME/src/main/resources/META-INF/plugin.xml`) — the official docs
confirm the element "can be skipped in the source plugin.xml file if the Gradle
plugin `patchPluginXml` task is enabled and configured" (Plugin Configuration File,
`idea-version` — see Reference). That page does not say what `patchPluginXml`
actually fills in, and it does not document the `pluginConfiguration { ideaVersion
{ ... } }` block you would use to override it — the only place either detail lives
is the Gradle plugin's own sources, and `source-lookup.md` rules out its Gradle
cache location as a citable source. Do not take either on faith: confirm the task
is still named `patchPluginXml` with `./gradlew tasks`, run it in the template, and
read the generated `build/tmp/patchPluginXml/plugin.xml` — the `<idea-version>` it injects
(and whether it carries an `until-build` at all) is the real, locally reproducible
answer for whatever platform dependency and Gradle plugin version the project
currently pins.

**Wrong:**

```kotlin
// range declared, but nothing in the release checklist ever runs the verifier
intellijPlatform {
    pluginConfiguration { ideaVersion { untilBuild.set(provider { null }) } }
}
```

**Right:**

```bash
./gradlew tasks | grep -i verif   # confirm the task name for this Gradle plugin version
./gradlew verifyPlugin            # IntelliJ Plugin Verifier; report under build/reports/pluginVerifier
```

Binding the verifier's IDE, and the multi-GB download it otherwise triggers, are a separate topic with their own
measured numbers — read [compat-verifier-ides-and-cost.md](compat-verifier-ides-and-cost.md) before the first
`verifyPlugin` run on a machine.

A version range also says nothing about which IDE **products** carry a given class (bundled plugins differ across IntelliJ
IDEA, PhpStorm, WebStorm — see [setup-dependency-origin.md](setup-dependency-origin.md) and AP-12), and the template's
verifier wiring (`.run/Run Verifications.run.xml` locally, `.github/workflows/build.yml` in CI) guarantees nothing by
existing: compatibility is never declared until that job has run and passed for the target IDE(s).

## Widening a range with no old IDE to verify it against

Critical rule 11 has no "but I disclosed it" exception: a `since-build` the verifier has never run
against is a claim, not a range. To verify a floor — or to widen one backwards with no old IDE
installed — work down this ladder and stop at the first rung affordable here, never past the last.

0. **The build you already compile against** — the resolved platform dependency is unpacked on disk
   as a complete IDE, so `local()` binds it at zero download cost. Unless `sinceBuild` was widened
   past it, it *is* the floor: [compat-verifier-ides-and-cost.md](compat-verifier-ides-and-cost.md).
1. **An old IDE already on the machine** — bind it, verify, done:
   `pluginVerification { ides { local("/path/to/old IDE.app") } }`. The IDE itself costs
   no download; [compat-verifier-ides-and-cost.md](compat-verifier-ides-and-cost.md) has
   what it still fetches. The run is also what proves every API the plugin calls exists
   at that floor.
2. **One named old build, fetched on purpose** — `ides { create(IntelliJPlatformType.IntellijIdeaCommunity, "2023.3") }`.
   The signature is `create(Any, String)`; confirm it in the resolved Gradle plugin jar
   with the `javap` recipe in `compat-verifier-ides-and-cost.md` rather than from memory.
   Name one build: `recommended()` and `select()` can match several releases and multiply
   the download. State the disk cost to the user before starting it.
3. **Neither is affordable** — then do not widen. Leave `sinceBuild` where the verifier has actually
   passed, ship that narrower range, and report the wider one as work not done. `plugin.xml` is what
   the Marketplace and the IDE read; a caveat in your report is not in `plugin.xml`.

Reference: `$PLUGIN_TEMPLATE_HOME/src/main/resources/META-INF/plugin.xml`,
`.run/Run Verifications.run.xml`, `.github/workflows/build.yml`;
`$IJ_SAMPLES/theme_basics/resources/META-INF/plugin.xml`;
[Plugin Configuration File — idea-version](https://plugins.jetbrains.com/docs/intellij/plugin-configuration-file.html#idea-plugin__idea-version).

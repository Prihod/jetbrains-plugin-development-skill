---
title: Raise the platform version, or migrate off Gradle IntelliJ Plugin 1.x
tags: workflow, compatibility, gradle, migration
verify: body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' references/workflow-upgrade-platform.md); for s in 'Record the current state (Step 0)**' 'Decide the new supported IDE range first — before touching versions.**' 'Raise the platform dependency**' 'Raise the Gradle plugin, Gradle, Kotlin**' 'Rebuild; fix compilation against the new API**' 'Replace APIs that became deprecated**' 'Run tests**' 'Run the verifier across the whole declared range**'; do printf '%s\n' "$body" | grep -qF -- "$s" || exit 1; done && printf '%s\n' "$body" | grep -qF -- 'not a version bump —' && printf '%s\n' "$body" | grep -qw -- 'AP-10' && printf '%s\n' "$body" | grep -qw -- 'AP-15' && grep -q '^## AP-10:' references/antipatterns-build.md && grep -q '^## AP-15:' references/antipatterns-api-deprecated.md
---

## Raise the platform version, or migrate off Gradle IntelliJ Plugin 1.x

Two different jobs share one procedure: raising `since`/`until` inside an already-current
build, and migrating a legacy `org.jetbrains.intellij` (1.x) project onto the
`org.jetbrains.intellij.platform` family `SKILL.md`'s Baseline table describes.

### Preconditions

- A plugin project that currently builds, tests and — if it already has a declared range — verifies clean; this changes an existing project, it does not create one. See [workflow-create-plugin.md](workflow-create-plugin.md) if there isn't one yet.
- `$PLUGIN_TEMPLATE_HOME` available, to compare current-baseline values against `SKILL.md`'s Baseline table.

### Steps

1. **Record the current state (Step 0)** — `SKILL.md`'s Step 0: platform version, `since`/`until`, Gradle, Kotlin, IntelliJ Platform Gradle Plugin id and version, dependencies. Without this snapshot there is nothing to diff against if the build breaks.
2. **Decide the new supported IDE range first — before touching versions.** The range drives every later choice — which deprecated APIs are actually reachable, which verifier targets to run; see [compat-range-and-verifier.md](compat-range-and-verifier.md).
3. **Raise the platform dependency** — `intellijIdea(...)` (or the equivalent platform-type call) inside `dependencies { intellijPlatform { } }`; see [setup-build.md](setup-build.md).
4. **Raise the Gradle plugin, Gradle, Kotlin** — the `version "..."` on `id("org.jetbrains.intellij.platform.settings")` and on the Kotlin JVM plugin id, both in `settings.gradle.kts`, plus the Gradle wrapper (`gradle/wrapper/gradle-wrapper.properties`); compare against the values a current template pins in `SKILL.md`'s Baseline table.
5. **Rebuild; fix compilation against the new API** — verify each broken call against sources before patching it, via [source-lookup.md](source-lookup.md)'s Finding sources order, not memory of the old signature.
6. **Replace APIs that became deprecated** — follow [compat-deprecated-policy.md](compat-deprecated-policy.md)'s procedure; do not silence the warning (AP-15).
7. **Run tests** — the levels and fixtures in [testing-levels-fixtures.md](testing-levels-fixtures.md), using the task names `./gradlew tasks` prints for this project.
8. **Run the verifier across the whole declared range** — [compat-range-and-verifier.md](compat-range-and-verifier.md); confirm the task name with `./gradlew tasks | grep -i verif` rather than assuming it is still the one this skill or an earlier project used.

Migrating off Gradle IntelliJ Plugin 1.x is not a version bump — it is a change of
build model: a different settings-plugin id (`org.jetbrains.intellij.platform.settings`
instead of `org.jetbrains.intellij`), a different dependency-declaration shape, and
several properties a current template no longer sets at all (AP-10 lists what
changed). Start it at Step 2, the same as any other upgrade — decide the supported
IDE range first, then adopt the new plugin id, never the other order.

### Validation

Never assume Gradle task names. List what this project actually offers first:

```
./gradlew tasks
```

Then run this project's build, test and verifier tasks, using exactly the names printed — not ones recalled from an earlier Gradle plugin version.

### Common mistakes

Treating "it compiles" as "it's supported" skips Step 8 — only that verifier run confirms the range in `plugin.xml` is real. AP-10 is the 1.x migration done as a version-number edit instead of a build-model change. AP-15 is Step 6 skipped in favor of `@Suppress("DEPRECATION")` on the newly-flagged calls.

### References

[compat-range-and-verifier.md](compat-range-and-verifier.md); [compat-deprecated-policy.md](compat-deprecated-policy.md); [setup-build.md](setup-build.md); [source-lookup.md](source-lookup.md); [testing-levels-fixtures.md](testing-levels-fixtures.md); [antipatterns-build.md](antipatterns-build.md); [antipatterns-api-deprecated.md](antipatterns-api-deprecated.md); [workflow-create-plugin.md](workflow-create-plugin.md); `SKILL.md`'s Baseline table.

---
title: Replace a deprecated API; when you can't, log a plan
tags: deprecated, api-status, logging, idea-log
verify: IJ_SRC="${IJ_SRC:?}"; PLUGIN_TEMPLATE_HOME="${PLUGIN_TEMPLATE_HOME:?}"; f=references/compat-deprecated-policy.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'getValues' || exit 1; printf '%s\n' "$body" | grep -qF 'getInstance' || exit 1; grep -q "getValues(@NonNls @NotNull String name)" "$IJ_SRC/platform/core-api/src/com/intellij/ide/util/PropertiesComponent.java" && grep -q "getInstance(@NotNull Class<?> cl)" "$IJ_SRC/platform/util/src/com/intellij/openapi/diagnostic/Logger.java" && grep -q "idea.log" "$PLUGIN_TEMPLATE_HOME/.run/Run Plugin.run.xml"
---

## Replace a deprecated API; when you can't, log a plan

Finding `@Deprecated` starts a procedure; suppressing the warning (AP-15) instead of
running it is how removed APIs end up shipped.

1. **Found a deprecated API** — `@Deprecated`, KDoc/javadoc `@deprecated`, or
   `@ApiStatus.ScheduledForRemoval` is the signal.
2. **Look for the replacement** — the javadoc usually names it directly:
   `PropertiesComponent.getValues(String)` (`PropertiesComponent.java:69`) says
   `@deprecated Use {@link #getList(String)}`, declared at `:77`.
3. **Check the target versions** — `@ApiStatus.ScheduledForRemoval(inVersion = "...")`
   gives a hard deadline, e.g. `IntellijSensitiveDataValidator.kt:169` carries
   `inVersion = "2026.1"`. Compare that deadline against the plugin's own
   `since`/`until` ([compat-range-and-verifier.md](compat-range-and-verifier.md)) —
   if removal falls inside the supported range, the old call already breaks part of
   the audience.
4. **Use the modern one.**

**Wrong:**

```kotlin
val tags: Array<String>? = properties.getValues("myPlugin.tags") // @Deprecated
```

**Right:**

```kotlin
val tags: List<String>? = properties.getList("myPlugin.tags")
```

If the deprecated call is genuinely unavoidable — no replacement covers this case
yet, or the replacement needs a platform version above the plugin's `since-build` —
record it next to the call, not only in a commit message:

```kotlin
// DEPRECATED-OK: PropertiesComponent.getValues, no List overload before build NNN.
// Affected: since-build..<NNN>. Alternative: getList(String).
// Removal plan: migrate once since-build is raised past NNN.
@Suppress("DEPRECATION")
val tags: Array<String>? = properties.getValues("myPlugin.tags")
```

## Logging and diagnosis

`Logger.getInstance(Class<?>)` (`Logger.java:114`; the category-string overload is
at `:110`) is the standard entry point — `Logger` is abstract, there is no
constructor to call.

Where the output lands depends on which run configuration launched the IDE; the
template's own `.run/` configs do not agree on the sandbox layout:
`Run Plugin.run.xml` and `Run Tests.run.xml` point at
`.intellijPlatform/sandbox/*/*/log{,-test}/idea.log`, while `Run Verifications.run.xml`
still points at the older `build/idea-sandbox/system/log/idea.log`. Treat both as
"search under the project's build/sandbox output," not as a fixed path.

Reference: `PropertiesComponent.java`, `Logger.java` in `intellij-community`
(`$IJ_SRC/platform/core-api/src/com/intellij/ide/util/`,
`$IJ_SRC/platform/util/src/com/intellij/openapi/diagnostic/`);
`platform/statistics/src/com/intellij/internal/statistic/eventLog/validator/IntellijSensitiveDataValidator.kt`;
`$PLUGIN_TEMPLATE_HOME/.run/Run Plugin.run.xml`, `Run Tests.run.xml`, `Run Verifications.run.xml`.

---
title: Load icons through IconLoader and strings through a resource bundle
tags: ui, icons, i18n, new-ui, resource-bundle
verify: IJ_SRC="${IJ_SRC:?}"; PLUGIN_TEMPLATE_HOME="${PLUGIN_TEMPLATE_HOME:?}"; f=references/ui-icons-i18n.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'getIcon(path: String, aClass: Class<*>): Icon' || exit 1; printf '%s\n' "$body" | grep -qF '<resource-bundle>messages.MyBundle</resource-bundle>' || exit 1; grep -n "fun getIcon(path: String, aClass: Class<\*>): Icon {" "$IJ_SRC/platform/util/ui/src/com/intellij/openapi/util/IconLoader.kt" && grep -n "<resource-bundle>messages.MyBundle</resource-bundle>" "$PLUGIN_TEMPLATE_HOME/src/main/resources/META-INF/plugin.xml"
---

## Load icons through IconLoader and strings through a resource bundle

Both traps look harmless until New UI or a second locale exposes them: a raw
`ImageIcon` bypasses caching, HiDPI scaling and the platform's automatic dark-theme
icon swap; a literal `String` label can't be localized without a recompile.

**Wrong (raw `ImageIcon`, literal label — from a real sample, `tool_window`):**

```java
label.setIcon(new ImageIcon(Objects.requireNonNull(getClass().getResource(imagePath))));
JButton refreshButton = new JButton("Refresh");
```

**Right (`IconLoader` for the icon, a bundle key for the label):**

```kotlin
val icon = IconLoader.getIcon("/icons/refresh.svg", javaClass)
JButton(MyBundle.message("action.refresh.text"))
```

`IconLoader.getIcon(path: String, aClass: Class<*>): Icon`
(`platform/util/ui/src/com/intellij/openapi/util/IconLoader.kt`) resolves the icon
relative to the caller class's classpath — the single-argument `getIcon(path: String)`
(`IconLoader.kt`) is `@Deprecated(level = DeprecationLevel.ERROR)` and will not
compile; always pass the class. The same loader also picks up a `<name>_dark.svg`
sibling automatically when the theme is dark — the `_dark.` suffix check lives at
`platform/util/ui/src/com/intellij/ui/icons/iconUtil.kt`. New UI icon sets are
selected through `ExperimentalUI` (`platform/core-ui/src/ui/ExperimentalUI.kt`,
abstract class) rather than by the plugin switching paths itself; ship one SVG (plus
`_dark` variant) and let the platform choose.

## Resource bundle and @Nls

`<resource-bundle>messages.MyBundle</resource-bundle>` in `plugin.xml`
(`$PLUGIN_TEMPLATE_HOME/src/main/resources/META-INF/plugin.xml`) points at a
`.properties` file; a bundle object extending `DynamicBundle`
(`platform/core-api/src/com/intellij/DynamicBundle.java`) exposes typed
`message(@PropertyKey(...) key, ...)` accessors — the template's `MyBundle.kt` is a
working instance of exactly this shape (`messages.MyBundle` in both files, matched).
Any user-facing `String` parameter meant to hold such a resolved, localized value is
typically annotated `@Nls` — e.g. `Configurable.focusOn(@NotNull @Nls String label)`
(`platform/ide-core/src/com/intellij/openapi/options/Configurable.java`); tooling
flags a literal passed where `@Nls` is expected. `@Nls` itself ships in the external
`org.jetbrains:annotations` jar, not in this `intellij-community` checkout, so it is
cited here by real usage sites rather than by its own declaration.

Reference: `platform/util/ui/src/com/intellij/openapi/util/IconLoader.kt`;
`platform/util/ui/src/com/intellij/ui/icons/iconUtil.kt`;
`platform/core-ui/src/ui/ExperimentalUI.kt`;
`platform/core-api/src/com/intellij/DynamicBundle.java`;
`platform/ide-core/src/com/intellij/openapi/options/Configurable.java`;
`$PLUGIN_TEMPLATE_HOME/src/main/resources/META-INF/plugin.xml`,
`src/main/kotlin/org/jetbrains/plugins/template/MyBundle.kt`;
`$IJ_SAMPLES/tool_window/src/main/java/org/intellij/sdk/toolWindow/CalendarToolWindowFactory.java`.

---
title: Build a dialog with the Kotlin UI DSL, and call init() yourself
tags: ui, dialogs, kotlin-ui-dsl, popups, notifications
verify: IJ_SRC="${IJ_SRC:?}"; f=references/ui-dsl-dialogs.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'DialogPanel' || exit 1; printf '%s\n' "$body" | grep -qF 'createCenterPanel' || exit 1; grep -n "fun panel(init: Panel.() -> Unit): DialogPanel" "$IJ_SRC/platform/platform-impl/src/com/intellij/ui/dsl/builder/builder.kt" && grep -n "abstract .*JComponent createCenterPanel();" "$IJ_SRC/platform/platform-api/src/com/intellij/openapi/ui/DialogWrapper.java"
---

## Build a dialog with the Kotlin UI DSL, and call init() yourself

`DialogWrapper` (`platform/platform-api/.../DialogWrapper.java`, abstract class) owns
the window chrome and button bar; your subclass supplies the body through
`createCenterPanel()` (`DialogWrapper.java`, `protected abstract @Nullable
JComponent createCenterPanel()`) and must call the protected `init()`
(`DialogWrapper.java`) itself — it is not called for you.

**Wrong (never calls `init()` — buttons and borders never get built):**

```kotlin
class MyDialog(project: Project) : DialogWrapper(project) {
    override fun createCenterPanel(): JComponent = panel {
        row("Name:") { textField() }
    }
    // missing: init()
}
```

**Right (call `init()` as the last statement of the constructor):**

```kotlin
class MyDialog(project: Project) : DialogWrapper(project) {
    init {
        title = "My Dialog"
        init() // wires createCenterPanel() and builds OK/Cancel
    }
    override fun createCenterPanel(): JComponent = panel {
        row("Name:") { textField() }
    }
}
```

A real subclass doing exactly this is `CommandLineDialog`
(`platform/external-system-impl/.../ui/command/line/CommandLineDialog.kt`): its
`init { }` block sets `title`, then ends with a bare `init()` call. Skip that call and
`showAndGet()` (`DialogWrapper.java`) still opens a window, just an empty one.

## The panel/row/cell builder

`panel { }` (`builder.kt`, returns a `DialogPanel`) is the entry point. Inside it,
`Panel.row(label, init)` (`Panel.kt`, two overloads — `row(label: String, …)` and
`row(label: JLabel? = null, …)`) adds a row; inside a row, `Row.cell(c)` (`Row.kt`,
`fun <T : JComponent> cell(component: T): Cell<T>`) wraps any Swing component as a
`Cell<T>`. `.align(align: Align)` (`Cell.kt`, values from `AlignX`/`AlignY` at
`Align.kt`) controls layout. `.bindText(...)` has a different overload per component
type — `Cell<C : JLabel>` at `label.kt`, and `Cell<T : JTextComponent>` (the one
[ui-settings.md](ui-settings.md)'s example actually calls) at `textField.kt`;
`.bindSelected(...)` (`button.kt`) is the checkbox form. A worked use, paired with
`BoundConfigurable` (`platform/platform-api/.../options/BoundConfigurable.kt`, see
[ui-settings.md](ui-settings.md)), lives in
`oauth2/src/main/kotlin/org.intellij.sdk.oauth2/AuthConfigurable.kt` (`oauth2` in
[source-lookup-samples.md](source-lookup-samples.md)):
`row { cell(content).align(AlignX.FILL) }`.

## Notifications and popups

For a message the user can dismiss without deciding anything, prefer a `Notification`
over another modal dialog: get a group with
`NotificationGroupManager.getInstance().getNotificationGroup(groupId)`
(`notification/NotificationGroupManager.java`) and construct a `Notification`
(`Notification.java`); the group is declared via the `notificationGroup` extension
point (`platform/ide-core-impl/resources/META-INF/IdeCore.xml`). For an anchored
popup instead of a window, use `JBPopupFactory`
(`platform/platform-api/.../ui/popup/JBPopupFactory.java`, static `getInstance()`).

Reference: `platform/platform-api/src/com/intellij/openapi/ui/DialogWrapper.java`;
`platform/platform-impl/src/com/intellij/ui/dsl/builder/builder.kt`,
`Panel.kt`, `Row.kt`, `Cell.kt`, `Align.kt`, `label.kt`, `textField.kt`, `button.kt`;
`platform/external-system-impl/src/com/intellij/openapi/externalSystem/service/ui/command/line/CommandLineDialog.kt`;
`platform/ide-core/src/com/intellij/notification/NotificationGroupManager.java`,
`Notification.java`; `platform/ide-core-impl/resources/META-INF/IdeCore.xml`;
`platform/platform-api/src/com/intellij/openapi/ui/popup/JBPopupFactory.java`;
`$IJ_SAMPLES/oauth2/src/main/kotlin/org.intellij.sdk.oauth2/AuthConfigurable.kt`.

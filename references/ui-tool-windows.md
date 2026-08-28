---
title: A tool window factory renders content; it does not hold business logic
tags: ui, tool-windows, services, layering
verify: IJ_SRC="${IJ_SRC:?}"; f=references/ui-tool-windows.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'PossiblyDumbAware' || exit 1; printf '%s\n' "$body" | grep -qF 'createToolWindowContent(project: Project, toolWindow: ToolWindow)' || exit 1; grep -n "interface ToolWindowFactory : PossiblyDumbAware" "$IJ_SRC/platform/platform-api/src/com/intellij/openapi/wm/ToolWindowFactory.kt" && grep -n "fun createToolWindowContent(project: Project, toolWindow: ToolWindow)" "$IJ_SRC/platform/platform-api/src/com/intellij/openapi/wm/ToolWindowFactory.kt"
---

## A tool window factory renders content; it does not hold business logic

`ToolWindowFactory` (`platform/platform-api/.../wm/ToolWindowFactory.kt:17`, interface
extending `PossiblyDumbAware`) builds a tool window's UI once, in
`@RequiresEdt fun createToolWindowContent(project: Project, toolWindow: ToolWindow)`
(`ToolWindowFactory.kt:33`). `shouldBeAvailable(project): Boolean` (`:54`, default
`true`) decides whether the stripe button even appears.

**Wrong (state and work live in the factory/content class itself):**

```kotlin
class MyToolWindowFactory : ToolWindowFactory {
    private var cachedResult: List<Item> = emptyList() // survives across re-opens? no owner.

    override fun createToolWindowContent(project: Project, toolWindow: ToolWindow) {
        cachedResult = expensiveScan(project) // domain work, done on the EDT
        val panel = JBPanel<Nothing>().apply { add(JBLabel(cachedResult.size.toString())) }
        toolWindow.contentManager.addContent(
            ContentFactory.getInstance().createContent(panel, null, false)
        )
    }
}
```

**Right (the factory only wires a service's data into a panel):**

```kotlin
class MyToolWindowFactory : ToolWindowFactory {
    override fun createToolWindowContent(project: Project, toolWindow: ToolWindow) {
        val service = project.service<MyProjectService>()
        val panel = JBPanel<Nothing>().apply { add(JBLabel(service.getRandomNumber().toString())) }
        toolWindow.contentManager.addContent(
            ContentFactory.getInstance().createContent(panel, null, false)
        )
    }
}
```

This is the general rule from [extend-services.md#layering](extend-services.md#layering)
applied to tool windows specifically: `UI → Service → Domain`, never the reverse. The
official `MyToolWindowFactory`
(`$PLUGIN_TEMPLATE_HOME/src/main/kotlin/org/jetbrains/plugins/template/toolWindow/MyToolWindowFactory.kt`)
follows exactly this shape — its `MyToolWindow` inner class reads
`toolWindow.project.service<MyProjectService>()` and never computes anything itself.
`tool_window`'s `CalendarToolWindowFactory` (`tool_window` in
[source-lookup-samples.md](source-lookup-samples.md)) is a second real factory,
`ContentFactory.getInstance().createContent(...)` (`ide-core/.../content/
ContentFactory.java:12`, interface) is how both attach a panel to the tool window.

## Lifecycle

Content needs a disposal owner too. `Content`
(`platform/ide-core/src/com/intellij/ui/content/Content.java:75,77`,
`getDisposer()`/`setDisposer(@NotNull Disposable disposer)`) is the hook:
`content.setDisposer(service)` disposes a `Disposable` — typically the backing
service — when the content is removed, not just when the project closes. Same
`Disposable`/`Disposer` contract as everywhere else, covered in full in
[lifecycle-disposable-messagebus.md](lifecycle-disposable-messagebus.md).

## Registration

`<toolWindow factoryClass="..." id="..."/>` registers against the platform's
`toolWindow` extension point — declared with `name="toolWindow"`
(`platform/platform-resources/src/META-INF/PlatformExtensionPoints.xml:207`,
`beanClass="com.intellij.openapi.wm.ToolWindowEP"`, `dynamic="true"`). `dynamic="true"`
means no IDE restart is required to pick up the registration; do not assume otherwise
when testing changes.

Reference: `platform/platform-api/src/com/intellij/openapi/wm/ToolWindowFactory.kt`;
`platform/ide-core/src/com/intellij/ui/content/ContentFactory.java`, `Content.java`;
`platform/platform-api/src/com/intellij/ui/components/JBLabel.java`,
`platform/util/ui/src/com/intellij/ui/components/JBPanel.java`;
`platform/platform-resources/src/META-INF/PlatformExtensionPoints.xml`; `$PLUGIN_TEMPLATE_HOME/src/main/kotlin/org/jetbrains/plugins/template/toolWindow/MyToolWindowFactory.kt`;
`$IJ_SAMPLES/tool_window/src/main/java/org/intellij/sdk/toolWindow/CalendarToolWindowFactory.java`.

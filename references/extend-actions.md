---
title: Add an action
tags: actions, action-system, threading
verify: IJ_SRC="${IJ_SRC:?}"; IJ_SAMPLES="${IJ_SAMPLES:?}"; F="$IJ_SRC/platform/ide-core/src/com/intellij/openapi/actionSystem/IdeActions.java"; test -f "$F" || exit 1; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' references/extend-actions.md); pairs=$(printf '%s\n' "$body" | sed -n 's/^| `\([A-Za-z]*\)` | `\(GROUP_[A-Z_]*\)` |.*/\2 \1/p'); test "$(printf '%s\n' "$pairs" | wc -l | tr -d ' ')" = 3 || exit 1; printf '%s\n' "$pairs" | while read -r c i; do grep -q "String $c = \"$i\";" "$F" || exit 1; done || exit 1; P="$IJ_SAMPLES/action_basics/src/main/resources/META-INF/plugin.xml"; grep -q 'group-id="EditorPopupMenu"' "$P" || exit 1; ! grep -q 'ProjectViewPopupMenu' "$P" || exit 1; grep -n 'ActionUpdateThread.BGT' "$IJ_SAMPLES/action_basics/src/main/java/org/intellij/sdk/action/CustomDefaultActionGroup.java"
---

## Add an action

An `AnAction` (or `ActionGroup`) must override `getActionUpdateThread()` explicitly;
left unset it silently runs `update()` on EDT — AP-02 — where `update()` must stay cheap.

**Wrong (no override — compiles, silently defaults to EDT, AP-02):**

```kotlin
class MyAction : AnAction() {
    override fun update(e: AnActionEvent) {
        val ref = e.getData(CommonDataKeys.PSI_ELEMENT) as? PsiReference
        e.presentation.isEnabledAndVisible = ref?.resolve() != null // PSI resolve on EDT — AP-03
    }
    override fun actionPerformed(e: AnActionEvent) { /* ... */ }
}
```

**Right:**

```kotlin
override fun getActionUpdateThread() = ActionUpdateThread.BGT
```

`BGT` is the platform's stated *preferred* value — it keeps the UI thread free — while
`EDT` is only the `default`, kept so simple actions don't have to think about it (see
the javadoc on `ActionUpdateThreadAware`). Declaring `BGT` explicitly:

- moves `update()` off the UI thread, so a PSI `resolve()` there stops freezing typing
  (AP-03 becomes structurally impossible, not just less likely);
- does not excuse a slow `update()` — background thread or not, it still runs on every
  action-system tick, so heavy work belongs in `actionPerformed()` or a service.

`actionPerformed()` delegates domain logic to a service — see
[extend-services.md#layering](extend-services.md#layering) — instead of doing the work
itself.

## Registering it in the right context menu

`add-to-group` needs a group id, and "the file's context menu" is three different menus.
The ids are constants in `platform/ide-core/src/com/intellij/openapi/actionSystem/IdeActions.java`:

| Group id | Constant | What it actually is | Installed at |
|---|---|---|---|
| `ProjectViewPopupMenu` | `GROUP_PROJECT_VIEW_POPUP` | right-click on a file or directory node in the Project tool window | `lang-impl/.../AbstractProjectViewPaneWithAsyncSupport.java:180` |
| `EditorPopupMenu` | `GROUP_EDITOR_POPUP` | right-click inside the editor text | `platform-impl/.../fileEditor/impl/text/TextEditorComponent.kt:74` |
| `EditorTabPopupMenu` | `GROUP_EDITOR_TAB_POPUP` | right-click on an editor tab | `platform-impl/.../fileEditor/impl/EditorTabbedContainer.kt:157` |

```xml
<action id="com.example.ShowFileInfo" class="com.example.ShowFileInfoAction" text="Show File Info">
    <add-to-group group-id="ProjectViewPopupMenu" anchor="first"/>
</action>
```

`$IJ_SAMPLES/action_basics` wires `EditorPopupMenu`, never `ProjectViewPopupMenu` — copying
that sample for a "file context menu" request lands the action on the editor menu instead.
The group also decides what the `DataContext` holds: the Project View pane's
`uiDataSnapshot` (`AbstractProjectViewPane.java:420`) supplies the tree selection —
`PlatformCoreDataKeys.SELECTED_ITEMS`, `CommonDataKeys.NAVIGATABLE_ARRAY`, a lazy
`CommonDataKeys.PSI_ELEMENT` — and no `CommonDataKeys.EDITOR`. Testing that context takes
a deliberate `DataContext`; see [testing-actions.md](testing-actions.md).

Reference: `platform/editor-ui-api/src/com/intellij/openapi/actionSystem/ActionUpdateThreadAware.java`;
`intellij-sdk-code-samples/action_basics/src/main/java/org/intellij/sdk/action/CustomDefaultActionGroup.java`.

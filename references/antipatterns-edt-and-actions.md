---
title: EDT and action-update-cycle traps
tags: threading, edt, action-system
verify: IJ_SRC="${IJ_SRC:?}"; f=references/antipatterns-edt-and-actions.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'platform/editor-ui-api/src/com/intellij/openapi/actionSystem/ActionUpdateThreadAware.java' || exit 1; sed -n '18,26p' "$IJ_SRC/platform/editor-ui-api/src/com/intellij/openapi/actionSystem/ActionUpdateThreadAware.java"
---

## AP-02: `getActionUpdateThread()` silently defaults to EDT

`ActionUpdateThreadAware.getActionUpdateThread()` has a `default` method returning
`ActionUpdateThread.EDT`. Code that never overrides it compiles and runs; the platform
javadoc's stated preference for `BGT` is invisible unless you go read it.

**Wrong:**

```kotlin
class MyAction : AnAction() {
    override fun update(e: AnActionEvent) { /* runs on EDT, unmarked */ }
}
```

**Right (add the override):**

```kotlin
override fun getActionUpdateThread() = ActionUpdateThread.BGT
```

**Caught by:** nothing

Reference: `platform/editor-ui-api/src/com/intellij/openapi/actionSystem/ActionUpdateThreadAware.java`

## AP-03: Resolving PSI inside `update()` while stuck on EDT

Because AP-02 defaults every action to EDT, `update()` blocks the UI thread when it
calls `resolve()` — a freeze that only shows up on real projects, not toy ones.

**Wrong:**

```kotlin
override fun update(e: AnActionEvent) {
    val target = (e.getData(CommonDataKeys.PSI_ELEMENT) as? PsiReference)?.resolve()
    e.presentation.isEnabledAndVisible = target != null
}
```

**Right (add the override, same as AP-02):**

```kotlin
override fun getActionUpdateThread() = ActionUpdateThread.BGT
```

**Caught by:** nothing

Reference: see AP-02; the same `ActionUpdateThreadAware.java`.

## AP-04: Slow I/O or heavy PSI work called directly on the EDT

Network calls, file I/O, or heavy PSI traversal invoked from an EDT context. Some
paths are checked by `SlowOperations` assertions, but plenty still slip through.

**Wrong:**

```kotlin
ApplicationManager.getApplication().invokeLater {
    val text = File(path).readText() // blocking I/O on the UI thread
}
```

**Right:**

```kotlin
ApplicationManager.getApplication().executeOnPooledThread {
    val text = File(path).readText()
    invokeLater { render(text) }
}
```

**Caught by:** runtime (`SlowOperations` assertions, only in some configurations)

Reference: `platform/core-api/src/com/intellij/util/SlowOperations.java`

---
title: Lifecycle and dumb-mode traps — resources and readiness you do not own
tags: lifecycle, disposable, dumb-mode, indexing
verify: IJ_SRC="${IJ_SRC:?}"; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' references/antipatterns-lifecycle-dumb-mode.md); norm=$(printf '%s' "$body" | tr '\n' ' ' | tr -s ' '); printf '%s' "$norm" | grep -qF 'Step 10 regression test is required' || exit 1; printf '%s' "$norm" | grep -qF '(workflow-test-and-debug.md)' || exit 1; grep -q '^10\. \*\*Regression test\*\*' references/workflow-test-and-debug.md || exit 1; ls "$IJ_SRC/platform/util/src/com/intellij/openapi/Disposable.java" "$IJ_SRC/platform/core-api/src/com/intellij/openapi/project/DumbAware.java"
---

## AP-06: A coroutine scope or subscription with no `Disposable` owner

A service launches a coroutine scope or subscribes to the message bus without a
`Disposable` owner. It compiles and runs, then leaks for the process lifetime once
the owning project or component is gone — there is no error, just an outlived scope.

**Wrong:**

```kotlin
class MyService(private val scope: CoroutineScope) {
    fun start() { scope.launch { pollForever() } } // never cancelled
}
```

**Right:**

```kotlin
class MyService(project: Project, private val scope: CoroutineScope) : Disposable {
    init { Disposer.register(project, this) }
    override fun dispose() { /* cancel or scope the coroutine to this Disposable */ }
}
```

**Caught by:** nothing

Reference: `platform/util/src/com/intellij/openapi/Disposable.java`

**Arriving here from a bug report?** A listener that outlives its owner, a callback that
fires after the project closed, a leaked scope — those are bugs, and the bug workflow
governs them: [workflow-test-and-debug.md](workflow-test-and-debug.md). Its Step 10
regression test is required, not optional; a lifecycle fix with no test that fails on the
pre-fix code has made the bug invisible, not gone.

## AP-14: Querying indices during dumb mode without `DumbAware`

An action or startup task queries indices (`findClass`, scope searches) without
`DumbAware`. It works after indexing finishes, then throws while reindexing.

**Wrong:**

```kotlin
class MyAction : AnAction() { // no DumbAware
    override fun actionPerformed(e: AnActionEvent) {
        JavaPsiFacade.getInstance(project).findClass(fqName, scope) // IndexNotReadyException
    }
}
```

**Right:**

```kotlin
class MyAction : AnAction(), DumbAware // or guard the call with DumbService.isDumb(project)
```

**Caught by:** runtime (`IndexNotReadyException`, only while the project is (re)indexing)

Reference: `platform/core-api/src/com/intellij/openapi/project/DumbAware.java`

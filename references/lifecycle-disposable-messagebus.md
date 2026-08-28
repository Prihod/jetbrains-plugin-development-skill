---
title: Own a lifecycle; subscribe and unsubscribe
tags: lifecycle, disposable, message-bus, disposer
verify: IJ_SRC="${IJ_SRC:?}"; HOME="${HOME:?}"; f=references/lifecycle-disposable-messagebus.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); norm=$(printf '%s' "$body" | tr '\n' ' ' | tr -s ' '); printf '%s' "$norm" | grep -qF 'Step 10 regression test is required' || exit 1; printf '%s' "$norm" | grep -qF '(workflow-test-and-debug.md)' || exit 1; for s in 'the `Disposable` passed to `connect(...)` decides when a connection stops hearing it' '`MessageBusImpl.connect` registers the connection against'; do printf '%s' "$norm" | grep -qF "$s" || exit 1; done; grep -q '^10\. \*\*Regression test\*\*' references/workflow-test-and-debug.md || exit 1; grep -q "fun connect(parentDisposable: Disposable): MessageBusConnection" "$IJ_SRC/platform/extensions/src/com/intellij/util/messages/MessageBus.kt" || exit 1; grep -A4 'override fun connect(parentDisposable: Disposable)' "$IJ_SRC/platform/core-api/src/com/intellij/util/messages/impl/MessageBusImpl.kt" | grep -q 'Disposer.register(parentDisposable, connection)' || exit 1; d=$(ls -d "$HOME"/.gradle/caches/*/transforms/*/transformed/idea*/ 2>/dev/null | head -1); test -n "$d" || exit 1; bt=$(find "$d" -maxdepth 2 -name build.txt | head -1); test -n "$bt" || exit 1; v=$(cat "$bt"); v=${v#*-}; TF=$(find "$HOME/.gradle/caches/modules-2/files-2.1/com.jetbrains.intellij.platform/test-framework/$v" -name "test-framework-$v.jar" | head -1); test -n "$TF" || exit 1; CP=$(ls "$d"lib/*.jar | tr '\n' ':'); base=$(printf '%s\n' "$body" | grep -oE ': [A-Za-z]+TestCase\(\)' | head -1); base=${base#: }; base=${base%()}; test -n "$base" || exit 1; javap -classpath "$TF" "com.intellij.testFramework.$base" | grep -q 'Project getProject();' || exit 1; m=$(printf '%s\n' "$body" | grep -oE 'PlatformTestUtil\.[A-Za-z]+\(project\)' | head -1); m=${m#PlatformTestUtil.}; m=${m%(project)}; test -n "$m" || exit 1; javap -classpath "$TF" com.intellij.testFramework.PlatformTestUtil | grep -q "void $m(com.intellij.openapi.project.Project)" || exit 1; sp=$(printf '%s\n' "$body" | grep -oE '\.[a-zA-Z]+\(VirtualFileManager\.[A-Z_]+\)' | head -1); t=${sp##*VirtualFileManager.}; t=${t%)}; sp=${sp%%(*}; sp=${sp#.}; test -n "$t" && test -n "$sp" || exit 1; javap -classpath "$CP" com.intellij.openapi.vfs.VirtualFileManager | grep -q " $t;" || exit 1; javap -classpath "$CP" com.intellij.util.messages.MessageBus | grep -q " $sp(com.intellij.util.messages.Topic" || exit 1; lm=$(printf '%s\n' "$body" | grep -oE '\)\.[a-z]+\(mutableListOf' | head -1); lm=${lm#).}; lm=${lm%(mutableListOf}; test -n "$lm" || exit 1; javap -classpath "$CP" com.intellij.openapi.vfs.newvfs.BulkFileListener | grep -q "void $lm(java.util.List" || exit 1
---

## Own a lifecycle; subscribe and unsubscribe

Every resource that must be cleaned up — a listener, a subscription, a coroutine scope
(AP-06) — needs exactly one `Disposable` owner. `Disposer.register(parent, child)`
attaches that owner to a longer-lived one (a project, the application, or another
`Disposable`), so `child.dispose()` runs when `parent` is disposed — no manual
"is closed" flag needed.

**Wrong (subscribe, own nothing that unsubscribes):**

```kotlin
project.messageBus.connect() // never disposed — outlives the code that created it
    .subscribe(MyTopic.TOPIC, MyListener())
```

**Right (tie the connection to a real owner):**

```kotlin
class MyProjectService(project: Project) : Disposable {
    init {
        project.messageBus.connect(this).subscribe(MyTopic.TOPIC, MyListener())
    }
    override fun dispose() { /* connection disposed automatically with `this` */ }
}
```

`MessageBus.connect(Disposable)` returns a `MessageBusConnection`, itself a
`Disposable`; disposing the owner disposes the connection, which unsubscribes.
`connect(CoroutineScope)` does the same tied to a scope instead — see
[threading-coroutines.md](threading-coroutines.md). A `Disposable` is never a lambda:
each instance needs object identity to sit correctly in the `Disposer` hierarchy.

**Message Bus is not the default choice.** It decouples publisher from subscriber
across the whole IDE — the right tool for a platform-wide event, wrong for two classes
that could just call a service method. Reach for `project.getMessageBus()` /
`ApplicationManager.getApplication().getMessageBus()` only when a plain service API
(see [extend-services.md](extend-services.md)) genuinely isn't enough — a broadcast to
unknown listeners, not a request/response between two known parties.

**Arriving here from a bug report?** A listener that outlives its owner, a callback that
fires after the project closed, a leaked scope — those are bugs, and the bug workflow
governs them: [workflow-test-and-debug.md](workflow-test-and-debug.md). Its Step 10
regression test is required, not optional; a lifecycle fix with no test that fails on the
pre-fix code has made the bug invisible, not gone.

**What that regression test looks like.** Only an Integration-level test can watch a project close —
a Light fixture reuses one project for the whole class and never closes it
([testing-levels-fixtures.md](testing-levels-fixtures.md)). Extend `HeavyPlatformTestCase`, close the
real project it opened with `PlatformTestUtil.forceCloseProjectWithoutSaving` (both in
`com.intellij.testFramework`), publish the topic, and assert the listener's own state did not move:

```kotlin
class ListenerDisposalTest : HeavyPlatformTestCase() {
    fun testListenerUnsubscribesWhenProjectCloses() {
        val service = project.service<MyProjectService>()
        val before = service.getCount()
        PlatformTestUtil.forceCloseProjectWithoutSaving(project)   // the real lifecycle event
        ApplicationManager.getApplication().messageBus
            .syncPublisher(VirtualFileManager.VFS_CHANGES).after(mutableListOf())
        assertEquals("listener fired after its project closed", before, service.getCount())
    }
}
```

Assert on state the listener changes, never on the connection object, and publish on the bus the
subscription was actually made on — which need not be the project's. The bus decides who hears an
event; the `Disposable` passed to `connect(...)` decides when a connection stops hearing it, because
that is the argument `MessageBusImpl.connect` registers the connection against. Run the test against
the reverted, pre-fix code first: a lifecycle test that also passes on the bug proves nothing.

Reference: `platform/util/src/com/intellij/openapi/Disposable.java`;
`platform/util/src/com/intellij/openapi/util/Disposer.java`;
`platform/extensions/src/com/intellij/util/messages/MessageBus.kt` and
`MessageBusConnection.kt`.

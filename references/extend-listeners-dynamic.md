---
title: Add a listener; keep the plugin dynamic
tags: listeners, message-bus, dynamic-plugins
verify: IJ_SAMPLES="${IJ_SAMPLES:?}"; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' references/extend-listeners-dynamic.md); norm=$(printf '%s' "$body" | tr '\n' ' ' | tr -s ' '); printf '%s' "$norm" | grep -qF 'Step 10 regression test is required' || exit 1; printf '%s' "$norm" | grep -qF '(workflow-test-and-debug.md)' || exit 1; grep -q '^10\. \*\*Regression test\*\*' references/workflow-test-and-debug.md || exit 1; grep -n 'applicationListeners' "$IJ_SAMPLES/max_opened_projects/src/main/resources/META-INF/plugin.xml"
---

## Add a listener; keep the plugin dynamic

A plugin can listen for platform events two ways — a declarative `<listener>` entry in
`plugin.xml`, or a programmatic `MessageBusConnection.subscribe(...)`. They are not
interchangeable: only the declarative form loads and unloads with the plugin for free.

**Wrong (subscribe programmatically, own nothing that disposes it):**

```kotlin
ApplicationManager.getApplication().messageBus.connect() // never disposed — leaks
    .subscribe(ProjectManager.TOPIC, MyListener())
```

`MessageBusConnection` itself implements `Disposable`; a connection with no owning
`Disposable` (or `CoroutineScope`) outlives the code that created it.

**Right (declare it; the platform owns the lifecycle):**

```xml
<applicationListeners>
    <listener class="org.intellij.sdk.maxOpenProjects.ProjectCloseListener"
              topic="com.intellij.openapi.project.ProjectManagerListener"/>
</applicationListeners>
```

```kotlin
class ProjectCloseListener : ProjectManagerListener {
    override fun projectClosed(project: Project) { /* ... */ }
}
```

If you must subscribe programmatically (e.g. narrower than application scope, inside a
service), pass the owning `Disposable` or `CoroutineScope` to `connect(...)` so
unsubscription is automatic:

```kotlin
messageBus.connect(parentDisposable).subscribe(TOPIC, listener)
```

**Arriving here from a bug report?** A listener that outlives its owner, a callback that
fires after the project closed, a leaked scope — those are bugs, and the bug workflow
governs them: [workflow-test-and-debug.md](workflow-test-and-debug.md). Its Step 10
regression test is required, not optional; a lifecycle fix with no test that fails on the
pre-fix code has made the bug invisible, not gone.

Reference: `platform/extensions/src/com/intellij/util/messages/MessageBusConnection.kt`;
`intellij-sdk-code-samples/max_opened_projects/src/main/resources/META-INF/plugin.xml`
and `.../ProjectCloseListener.java`.

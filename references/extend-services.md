---
title: Add a service; keep logic out of the UI
tags: services, application, project, layering
verify: IJ_SAMPLES="${IJ_SAMPLES:?}"; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' references/extend-services.md); norm=$(printf '%s' "$body" | tr '\n' ' ' | tr -s ' '); printf '%s' "$norm" | grep -qF 'Step 10 regression test is required' || exit 1; printf '%s' "$norm" | grep -qF '(workflow-test-and-debug.md)' || exit 1; grep -q '^10\. \*\*Regression test\*\*' references/workflow-test-and-debug.md || exit 1; grep -n '@Service' "$IJ_SAMPLES/max_opened_projects/src/main/java/org/intellij/sdk/maxOpenProjects/ProjectCountingService.java"
---

## Add a service; keep logic out of the UI

A service is the platform's DI-managed singleton — application- or project-scoped —
and it is where domain logic belongs, not inside an action, a tool window, or a dialog.

**Wrong (looked up through the deprecated container API — AP-01):**

```kotlin
val settings = ServiceManager.getService(MySettings::class.java)
```

**Right (light service; default scope is `Level.APP`, or declare `Level.PROJECT`):**

```kotlin
@Service(Service.Level.PROJECT)
class MyProjectService(private val project: Project) {
    fun doWork() { /* domain logic lives here, not in the caller */ }
}
```

```kotlin
val service = project.getService(MyProjectService::class.java) // or Application.getService(...)
```

A light service like this needs no XML registration at all. `max_opened_projects`'
`ProjectCountingService` is a real, bare `@Service`-annotated final class, retrieved
with `ApplicationManager.getApplication().getService(...)` from a listener.

**Disposal.** A service that owns a resource needing explicit cleanup — a coroutine
scope, a message-bus subscription, a listener registration — implements `Disposable`:

```kotlin
@Service(Service.Level.PROJECT)
class MyProjectService(project: Project) : Disposable {
    private val scope = /* ... */
    override fun dispose() { /* cancel the scope, close the resource */ }
}
```

No manual `Disposer.register(project, this)` call is needed for this base case: the
container registers any service instance implementing `Disposable` against its own
internal parent disposable at creation time, and disposes it when the owning
application or project is disposed. A service that holds such a resource without
implementing `Disposable` is exactly AP-06's leak. For the full `Disposable`
contract and when a message-bus subscription needs one, see
[lifecycle-disposable-messagebus.md](lifecycle-disposable-messagebus.md).

## Layering

```
UI → Action / ToolWindow / Editor integration → Service → Domain → Platform API
```

An action, tool window, or dialog calls a service; the service may call a plain domain
class; only the service or domain layer touches the platform API directly. Nothing
above the service layer holds business state, and a service is not reached for as a
generic global-singleton shortcut when a narrower owner already exists.

**Arriving here from a bug report?** A service that leaks a listener or a scope is a bug, and the bug workflow governs it: [workflow-test-and-debug.md](workflow-test-and-debug.md) — its Step 10 regression test is required, not optional.

Reference: `platform/core-api/src/com/intellij/openapi/components/Service.java`;
`platform/service-container/src/com/intellij/serviceContainer/containerUtil.kt`
(`initializeComponentOrLightService`, auto-registers `Disposable` services);
`intellij-sdk-code-samples/max_opened_projects/src/main/java/org/intellij/sdk/maxOpenProjects/ProjectCountingService.java`.

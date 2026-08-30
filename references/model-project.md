---
title: Project, Module and roots — and projects that are already gone
tags: project, module, roots, disposed
verify: IJ_SRC="${IJ_SRC:?}"; f=references/model-project.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'ComponentManager' || exit 1; printf '%s\n' "$body" | grep -qF 'AreaInstance' || exit 1; grep -n "public interface Project extends ComponentManager, AreaInstance" "$IJ_SRC/platform/core-api/src/com/intellij/openapi/project/Project.java"
---

## Project, Module and roots

`Project` (`platform/core-api/src/com/intellij/openapi/project/Project.java`, an
interface extending `ComponentManager` and `AreaInstance`) is the top-level container a
plugin is handed almost everywhere — services, actions, listeners. `Module`
(`platform/core-api/src/com/intellij/openapi/module/Module.java`, extending
`ComponentManager`, `AreaInstance`, `Disposable`) is one unit inside it; a project holds
zero or more modules. `ComponentManager`
(`platform/extensions/src/com/intellij/openapi/components/ComponentManager.java`)
is what supplies `isDisposed()` to both.

**Enumerating modules and roots.**

```kotlin
val modules = ModuleManager.getInstance(project).modules // platform/projectModel-api .../ModuleManager.kt
val contentRoots = ProjectRootManager.getInstance(project).contentRoots // .../ProjectRootManager.java
val moduleRoots = ModuleRootManager.getInstance(module) // .../ModuleRootManager.java
val sourceRoots = moduleRoots.sourceRoots // .../ModuleRootModel.java
```

`ProjectRootManager` answers project-wide root questions; `ModuleRootManager`
(via `ModuleRootModel`) answers them per module — `getContentRoots()` and
`getSourceRoots()` both return `VirtualFile[]`, so treat them through
[model-vfs.md](model-vfs.md) once you have them, not through `java.io.File`.

**Disposed projects.** A `Project` reference can outlive the project it points to —
held in a listener, a cached field, a coroutine that was slow to finish. Because
`Project` inherits `isDisposed()` from `ComponentManager`, check it before doing
anything with a project you didn't just receive as a fresh parameter:

**Wrong (cached project reference, no disposal check):**

```kotlin
class MyListener(private val project: Project) { // stored for later use
    fun onEvent() {
        project.getService(MyProjectService::class.java).handle() // AlreadyDisposedException once project is closed
    }
}
```

**Right — check before touching services, PSI or the VFS through it:**

```kotlin
class MyListener(private val project: Project) {
    fun onEvent() {
        if (project.isDisposed) return
        project.getService(MyProjectService::class.java).handle()
    }
}
```

This is the same category of bug as AP-06 (a resource with no `Disposable` owner),
approached from the consumer side. Prefer structuring code so it is cancelled when the
project closes (see
[lifecycle-disposable-messagebus.md](lifecycle-disposable-messagebus.md)) over adding
`isDisposed` checks everywhere as a substitute for that.

For a worked example of the project/module model — SDKs, libraries, project structure —
see `project_model` in `$IJ_SAMPLES`
([source-lookup-samples.md](source-lookup-samples.md)).

Reference: `platform/core-api/src/com/intellij/openapi/project/Project.java`;
`platform/core-api/src/com/intellij/openapi/module/Module.java`;
`platform/extensions/src/com/intellij/openapi/components/ComponentManager.java`;
`platform/core-api/src/com/intellij/serviceContainer/AlreadyDisposedException.java`;
`platform/projectModel-api/src/com/intellij/openapi/module/ModuleManager.kt`;
`platform/projectModel-api/src/com/intellij/openapi/roots/ProjectRootManager.java`;
`platform/projectModel-api/src/com/intellij/openapi/roots/ModuleRootManager.java`;
`platform/projectModel-api/src/com/intellij/openapi/roots/ModuleRootModel.java`.

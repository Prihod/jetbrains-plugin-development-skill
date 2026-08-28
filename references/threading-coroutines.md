---
title: Use coroutines without leaking scopes
tags: threading, coroutines, structured-concurrency, disposable
verify: HOME="${HOME:?}"; f=references/threading-coroutines.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); d=$(ls -d "$HOME"/.gradle/caches/*/transforms/*/transformed/ideaIU-2025.2.6.2 2>/dev/null | head -1); test -n "$d" || exit 1; bt=$(find "$d" -maxdepth 2 -name build.txt | head -1); test -n "$bt" || exit 1; real=$(cat "$bt"); j="$d/lib/app-client.jar"; app='expected (), (CoroutineScope), (Application), or (Application, CoroutineScope)'; prj='expected (Project), (Project, CoroutineScope), (CoroutineScope), or ()'; mod='expected (Module) or ()'; unzip -p "$j" com/intellij/serviceContainer/ComponentManagerImpl.class | LC_ALL=C grep -qF "$app" || exit 1; unzip -p "$j" com/intellij/openapi/project/impl/ProjectImpl.class | LC_ALL=C grep -qF "$prj" || exit 1; unzip -p "$j" com/intellij/openapi/module/impl/ModuleComponentManager.class | LC_ALL=C grep -qF "$mod" || exit 1; norm=$(printf '%s' "$body" | tr '\n' ' ' | tr -s ' '); for s in '`()`, `(CoroutineScope)`, `(Application)`, `(Application, CoroutineScope)`' '`()`, `(CoroutineScope)`, `(Project)`, `(Project, CoroutineScope)`' '`()`, `(Module)`' "$prj"; do printf '%s' "$norm" | grep -qF "$s" || exit 1; done; printf '%s' "$norm" | grep -qF "$real" || exit 1; for x in $(printf '%s' "$norm" | grep -oE '[A-Z]{2}-[0-9]+\.[0-9]+\.[0-9]+'); do test "$x" = "$real" || exit 1; done
---

## Use coroutines without leaking scopes

A `CoroutineScope` is a resource like any other: something must own it and cancel it
when that owner goes away. The container hands a service one for free — take that
instead of building a private, unowned one.

**Wrong (owns nothing, cancels nothing — AP-06):**

```kotlin
class MyService {
    private val scope = CoroutineScope(SupervisorJob()) // no owner, never canceled
    fun start() { scope.launch { pollForever() } }
}
```

**Right (constructor-injected scope; the container cancels it on disposal):**

```kotlin
@Service(Service.Level.PROJECT)
class MyService(project: Project, private val scope: CoroutineScope) {
    fun start() { scope.launch { pollForever() } }
}
```

**Which constructor shapes are accepted depends on the container**, not on one
platform-wide list: `ProjectImpl` overrides the lookup `ComponentManagerImpl` defines,
and the module container overrides it again. At the Baseline's target build
`IU-252.28539.54`:

| Level — container doing the lookup | Accepted constructor shapes |
|---|---|
| `APP` — `ComponentManagerImpl` | `()`, `(CoroutineScope)`, `(Application)`, `(Application, CoroutineScope)` |
| `PROJECT` — `ProjectImpl` | `()`, `(CoroutineScope)`, `(Project)`, `(Project, CoroutineScope)` |
| module — `ModuleComponentManager` | `()`, `(Module)` — no scope is injected at all |

So `(Project, CoroutineScope)` in the example above is the project-level list, not a
fifth application-level shape; and a module-level service that wants a scope has to get
one some other way. The container looks up exactly its own level's shapes by reflection
and injects a scope it owns and cancels together with the service, so no manual
`Disposer.register` is needed for that scope specifically. Read your level's list out
of the build you target — each container spells it out in the message it throws:

```bash
DIST=$(ls -d ~/.gradle/caches/*/transforms/*/transformed/ideaIU-*/ | head -1)
unzip -p "$DIST/lib/app-client.jar" \
  com/intellij/openapi/project/impl/ProjectImpl.class |
  strings | grep 'Cannot find suitable constructor'
# Cannot find suitable constructor, expected (Project), (Project, CoroutineScope), (CoroutineScope), or ()
```

Narrower work than the service's own lifetime gets a named child instead of reusing
the injected scope directly, so it can be canceled early without tearing down the
service:

```kotlin
val requestScope = scope.childScope("MyService.request")
```

`childScope` is canceled automatically when its parent is, but a child with a
genuinely shorter lifetime (e.g. one per request) must still be canceled explicitly —
otherwise it lives until the parent does, which is the same shape of leak as AP-06 on
a smaller scale.

Structured concurrency means every `launch`/`async` has a scope with a clear parent;
nothing calls `GlobalScope.launch` or builds a bare `CoroutineScope()` with no
registered owner. For subscribing to the message bus from inside a coroutine, see
[lifecycle-disposable-messagebus.md](lifecycle-disposable-messagebus.md).

Reference: `com/intellij/serviceContainer/ComponentManagerImpl.class`,
`com/intellij/openapi/project/impl/ProjectImpl.class` and
`com/intellij/openapi/module/impl/ModuleComponentManager.class` inside
`lib/app-client.jar` of the resolved `IU-252.28539.54` distribution
(`findConstructorAndInstantiateClass`); `platform/util/coroutines/src/coroutineScope.kt`
(`childScope`).

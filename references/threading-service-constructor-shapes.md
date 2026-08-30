---
title: Pick a constructor the container can call
tags: services, coroutines, constructor, scope-injection, containers
verify: HOME="${HOME:?}"; f=references/threading-service-constructor-shapes.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); d=$(ls -d "$HOME"/.gradle/caches/*/transforms/*/transformed/ideaIU-2025.2.6.2 2>/dev/null | head -1); test -n "$d" || exit 1; bt=$(find "$d" -maxdepth 2 -name build.txt | head -1); test -n "$bt" || exit 1; real=$(cat "$bt"); j="$d/lib/app-client.jar"; app='expected (), (CoroutineScope), (Application), or (Application, CoroutineScope)'; prj='expected (Project), (Project, CoroutineScope), (CoroutineScope), or ()'; dprj='expected (Project) or ()'; mod='expected (Module) or ()'; unzip -p "$j" com/intellij/serviceContainer/ComponentManagerImpl.class | LC_ALL=C grep -qF "$app" || exit 1; unzip -p "$j" com/intellij/openapi/project/impl/ProjectImpl.class | LC_ALL=C grep -qF "$prj" || exit 1; unzip -p "$j" com/intellij/openapi/project/impl/DefaultProjectImpl.class | LC_ALL=C grep -qF "$dprj" || exit 1; unzip -p "$j" com/intellij/openapi/module/impl/ModuleComponentManager.class | LC_ALL=C grep -qF "$mod" || exit 1; norm=$(printf '%s' "$body" | tr '\n' ' ' | tr -s ' '); for s in '`()`, `(CoroutineScope)`, `(Application)`, `(Application, CoroutineScope)`' '`()`, `(CoroutineScope)`, `(Project)`, `(Project, CoroutineScope)`' '`(Project)`, `()` — no scope' '`()`, `(Module)` — no scope' "$prj" "$dprj"; do printf '%s' "$norm" | grep -qF "$s" || exit 1; done; printf '%s' "$norm" | grep -qF "$real" || exit 1; for x in $(printf '%s' "$norm" | grep -oE '[A-Z]{2}-[0-9]+\.[0-9]+\.[0-9]+'); do test "$x" = "$real" || exit 1; done
---

## Pick a constructor the container can call

A service is instantiated by reflection: the container looks for a constructor whose
parameters match one of the shapes **it** accepts, and throws if none does. The list is
not platform-wide. `ProjectImpl` overrides the lookup `ComponentManagerImpl` defines,
the default project overrides it again, and so does the module container. At the
Baseline's target build `IU-252.28539.54`:

| Level — container doing the lookup | Accepted constructor shapes |
|---|---|
| `APP` — `ComponentManagerImpl` | `()`, `(CoroutineScope)`, `(Application)`, `(Application, CoroutineScope)` |
| `PROJECT` — `ProjectImpl` | `()`, `(CoroutineScope)`, `(Project)`, `(Project, CoroutineScope)` |
| default project — `DefaultProjectImpl` | `(Project)`, `()` — no scope is injected |
| module — `ModuleComponentManager` | `()`, `(Module)` — no scope is injected |

Two consequences worth planning around, both read straight off the messages below:

- **The default project is not the project container.** It backs the settings template
  for new projects, and its list has no `CoroutineScope` shape at all. A project-level
  service whose only constructor takes a scope therefore cannot be instantiated there,
  even though the same class constructs fine in a real project.
- **A module-level service gets no scope either** and has to obtain one some other way.

So `(Project, CoroutineScope)` is the project-level list, not a fifth application-level
shape. The scope a container does inject is one it owns and cancels together with the
service, so no manual `Disposer.register` is needed for that scope specifically — see
[threading-coroutines.md](threading-coroutines.md) for what to do with it.

Read your level's list out of the build you target rather than from this table: each
container spells it out in the message it throws.

```bash
DIST=$(ls -d ~/.gradle/caches/*/transforms/*/transformed/ideaIU-*/ | head -1)
for c in openapi/project/impl/ProjectImpl openapi/project/impl/DefaultProjectImpl; do
  unzip -p "$DIST/lib/app-client.jar" "com/intellij/$c.class" |
    strings | grep 'Cannot find suitable constructor'
done
# Cannot find suitable constructor, expected (Project), (Project, CoroutineScope), (CoroutineScope), or ()
# Cannot find suitable constructor, expected (Project) or ()
```

The same command reads the application and module lists from
`com/intellij/serviceContainer/ComponentManagerImpl.class` and
`com/intellij/openapi/module/impl/ModuleComponentManager.class`.

Reference: `com/intellij/serviceContainer/ComponentManagerImpl.class`,
`com/intellij/openapi/project/impl/ProjectImpl.class`,
`com/intellij/openapi/project/impl/DefaultProjectImpl.class` and
`com/intellij/openapi/module/impl/ModuleComponentManager.class` inside
`lib/app-client.jar` of the resolved `IU-252.28539.54` distribution
(`findConstructorAndInstantiateClass`).

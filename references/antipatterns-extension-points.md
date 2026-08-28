---
title: Extension point traps — dynamic loading and reinvented registries
tags: extension-points, dynamic-plugins, plugin-xml
verify: IJ_SRC="${IJ_SRC:?}"; f=references/antipatterns-extension-points.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'dynamic="true"' || exit 1; grep -n 'dynamic="true"' "$IJ_SRC/platform/navbar/backend/resources/intellij.platform.navbar.backend.xml"
---

## AP-08: Declaring an extension point without `dynamic="true"`

The EP compiles and works. The defect is that plugins registering against it force a
full IDE restart on install, update, or uninstall instead of unloading cleanly — visible
only during plugin verification or in production, never while coding.

**Wrong:**

```xml
<extensionPoint qualifiedName="my.plugin.myExtension" interface="my.plugin.MyExtension"/>
```

**Right:**

```xml
<extensionPoint qualifiedName="my.plugin.myExtension" interface="my.plugin.MyExtension" dynamic="true"/>
```

**Caught by:** nothing (surfaces as a forced restart, or a marketplace verifier note)

Reference: e.g. `platform/navbar/backend/resources/intellij.platform.navbar.backend.xml`

## AP-13: A custom extension point where the platform already has one

Writing a private registry or SPI for pluggable behavior instead of reusing the
platform's own extension point. It compiles and runs — it just duplicates a mechanism
the platform already provides, and stays invisible to tooling built around real EPs.

**Wrong:**

```kotlin
object MyRegistry { val handlers = mutableListOf<Handler>() } // reinvents an EP
```

**Right:**

```xml
<extensionPoint qualifiedName="my.plugin.handler" interface="my.plugin.Handler" dynamic="true"/>
```

**Caught by:** nothing (both compile and run identically well)

Reference: check first with [source-lookup.md#search-recipes](source-lookup.md#search-recipes) — search
for an existing `<extensionPoint>` before writing a new registry.

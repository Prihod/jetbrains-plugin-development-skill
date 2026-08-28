---
title: Deprecated and legacy platform API traps
tags: api, deprecated, service, legacy-components
verify: IJ_SRC="${IJ_SRC:?}"; f=references/antipatterns-api-deprecated.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'platform/core-api/src/com/intellij/openapi/components/ServiceManager.java' || exit 1; sed -n '1,16p' "$IJ_SRC/platform/core-api/src/com/intellij/openapi/components/ServiceManager.java"
---

## AP-01: `ServiceManager.getService(...)` instead of the container's own method

`ServiceManager` is `@Deprecated` with the javadoc `Don't use.`; its `getServiceIfCreated`
is additionally `@ApiStatus.ScheduledForRemoval`. It still returns a working service
today, so the deprecation strikethrough is the only warning you get.

**Wrong:**

```kotlin
val settings = ServiceManager.getService(MySettings::class.java)
```

**Right:**

```kotlin
val settings = project.getService(MySettings::class.java) // or Application.getService(...)
```

**Caught by:** compiler (deprecation warning; the build still succeeds)

Reference: `platform/core-api/src/com/intellij/openapi/components/ServiceManager.java`

## AP-07: `ProjectComponent` / `ApplicationComponent` instead of a service

Both interfaces are `@Deprecated`, part of the legacy component model. Plugins that
declare them don't support dynamic loading — but the class still compiles and runs.

**Wrong (`plugin.xml`):**

```xml
<project-components>
  <component><implementation-class>my.plugin.MyComponent</implementation-class></component>
</project-components>
```

**Right:**

```kotlin
@Service(Service.Level.PROJECT)
class MyService(project: Project)
```

**Caught by:** compiler (deprecation warning on the interface)

Reference: `platform/core-api/src/com/intellij/openapi/components/ProjectComponent.java`

## AP-15: `@Suppress("DEPRECATION")` instead of migrating

Suppressing the warning removes the one signal that a deprecated, soon-to-be-removed
API is in use. The code keeps compiling cleanly right up until the API is deleted.

**Wrong:**

```kotlin
@Suppress("DEPRECATION")
val settings = ServiceManager.getService(MySettings::class.java)
```

**Right:**

```kotlin
val settings = project.getService(MySettings::class.java) // migrate instead of silencing
```

**Caught by:** nothing (that is the point of the suppression)

Reference: `platform/core-api/src/com/intellij/openapi/components/ServiceManager.java`

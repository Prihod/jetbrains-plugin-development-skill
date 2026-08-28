---
title: PSI and VFS traps — reaching past the IDE's model of code and files
tags: psi, vfs, read-action
verify: IJ_SRC="${IJ_SRC:?}"; f=references/antipatterns-psi-vfs.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'PsiInvalidElementAccessException' || exit 1; find "$IJ_SRC/platform/core-api/src/com/intellij/psi" -name "PsiInvalidElementAccessException.java" | grep .
---

## AP-05: Caching a `PsiElement` across read actions

A `PsiElement` reference is stored and reused after the document changes. The element
can become invalid; using it throws `PsiInvalidElementAccessException` — but only at
the later access point, far from where it was originally cached.

**Wrong:**

```kotlin
val cached: PsiElement = resolveTarget() // stored on the instance
fun later() = cached.text // may throw PsiInvalidElementAccessException
```

**Right:**

```kotlin
val pointer = SmartPointerManager.createPointer(resolveTarget())
fun later() = pointer.element?.text // null once invalid, never throws
```

**Caught by:** runtime (`PsiInvalidElementAccessException`, only when the stale reference is touched)

Reference: `platform/core-api/src/com/intellij/psi/PsiInvalidElementAccessException.java`

## AP-11: `java.io.File` instead of the VFS for project files

Reading or writing project files with `java.io.File`/`java.nio` bypasses
`VirtualFile`. The write succeeds on disk, but the IDE's in-memory model doesn't see
it until an explicit refresh — editors and indices quietly show stale content.

**Wrong:**

```kotlin
File(path).writeText(newContent) // IDE model doesn't know this happened
```

**Right:**

```kotlin
VfsUtil.saveText(virtualFile, newContent) // or virtualFile.refresh(...) after external writes
```

**Caught by:** nothing (the IDE just shows stale state; no exception, no warning)

Reference: `platform/core-api/src/com/intellij/openapi/vfs/VirtualFile.java`

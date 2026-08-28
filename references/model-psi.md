---
title: Choose the right layer of the code model — Text, Document, PSI, Stub/Index
tags: psi, model, dumb-mode, indexing
verify: IJ_SRC="${IJ_SRC:?}"; f=references/model-psi.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'PsiFileSystemItem' || exit 1; grep -n "public interface PsiFile extends PsiFileSystemItem" "$IJ_SRC/platform/core-api/src/com/intellij/psi/PsiFile.java"
---

## Choose the right layer of the code model

The platform represents a file at four layers. Pick the lowest one that answers the
question — climbing higher costs a parse (or an index rebuild) you may not need:

```
Text       -> raw characters, no structure
Document   -> mutable text with offsets, what the editor edits
PSI        -> parsed tree with semantics
Stub/Index -> what you query without parsing every file
```

Cursor offsets, line/column math, plain text edits: stay at **Document**. Resolving a
reference, renaming a symbol, understanding structure: go to **PSI**. Answering "which
files declare a class named X" across a whole project without opening each one: use
**Stub/Index**.

**PSI.** `PsiFile` (`platform/core-api/src/com/intellij/psi/PsiFile.java:26`, extends
`PsiFileSystemItem`) and `PsiElement`
(`platform/core-api/src/com/intellij/psi/PsiElement.java:35`, extends `UserDataHolder`,
`Iconable`) form the parsed tree. `PsiManager.findFile(VirtualFile)` gets the `PsiFile`
for a `VirtualFile`; `PsiDocumentManager.getPsiFile(Document)` /
`.getDocument(PsiFile)` cross between the Document and PSI layers, and
`commitDocument(Document)` flushes pending Document edits into PSI before you read it.

Every read of PSI needs a read action and every mutation a write action on the EDT —
that contract, `ReadAction.nonBlocking`, and the reason `PsiElement` references go
stale are covered in full in
[threading-read-write.md](threading-read-write.md); this file does not repeat them.
Caching a raw `PsiElement` across two read actions is AP-05 — use a
`SmartPsiElementPointer` instead.

**Dumb mode.** While the platform is (re)indexing, `DumbService.isDumb(project)` is
`true` and most index-backed lookups throw `IndexNotReadyException`
(`platform/core-api/src/com/intellij/openapi/project/IndexNotReadyException.java`)
unless the caller is `DumbAware`
(`platform/core-api/src/com/intellij/openapi/project/DumbAware.java:26`). Querying an
index with no dumb-mode guard is AP-14:

**Wrong (no dumb-mode guard):**

```kotlin
fun resolve(project: Project, fqName: String, scope: GlobalSearchScope) =
    JavaPsiFacade.getInstance(project).findClass(fqName, scope) // IndexNotReadyException while (re)indexing
```

**Right — guard, or defer until indexing finishes:**

```kotlin
fun resolve(project: Project, fqName: String, scope: GlobalSearchScope): PsiClass? {
    if (DumbService.isDumb(project)) return null // or DumbService.runWhenSmart { resolve(...) }
    return JavaPsiFacade.getInstance(project).findClass(fqName, scope)
}
```

**Stub/Index.** `StubIndex.getInstance().getElements(key, ...)`
(`platform/indexing-api/src/com/intellij/psi/stubs/StubIndex.java:36,93`) and
`FileBasedIndex.getInstance().getValues(id, key, scope)`
(`platform/indexing-api/src/com/intellij/util/indexing/FileBasedIndex.java:126,148`)
answer "which elements/files match this key" from a prebuilt index, without parsing
every candidate file — this is what makes project-wide search and completion tractable.

For a worked PSI walk, see `psi_demo` in `$IJ_SAMPLES` (mapped in
[source-lookup-samples.md](source-lookup-samples.md)).

Reference: `platform/core-api/src/com/intellij/psi/PsiFile.java`;
`platform/core-api/src/com/intellij/psi/PsiElement.java`;
`platform/core-api/src/com/intellij/psi/PsiManager.java` (`findFile`);
`platform/core-api/src/com/intellij/psi/PsiDocumentManager.java`;
`platform/core-api/src/com/intellij/openapi/project/DumbService.kt`;
`java/java-psi-api/src/com/intellij/psi/JavaPsiFacade.java` (`getInstance`, `findClass`);
`platform/indexing-api/src/com/intellij/psi/stubs/StubIndex.java`;
`platform/indexing-api/src/com/intellij/util/indexing/FileBasedIndex.java`.

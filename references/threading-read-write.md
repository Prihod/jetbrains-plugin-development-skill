---
title: Read or write the code model safely
tags: threading, read-action, write-action, psi
verify: IJ_SRC="${IJ_SRC:?}"; HOME="${HOME:?}"; f=references/threading-read-write.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'ReadAction.compute<PsiElement?, Throwable>' || exit 1; printf '%s\n' "$body" | grep -qF 'computeBlocking' || exit 1; d=$(ls -d "$HOME"/.gradle/caches/*/transforms/*/transformed/ideaIU-2025.2.6.2 2>/dev/null | head -1); test -n "$d" || exit 1; bt=$(find "$d" -maxdepth 2 -name build.txt | head -1); test -n "$bt" || exit 1; grep -q '^IU-252\.' "$bt" || exit 1; api=$(javap -classpath "$d/lib/util-8.jar" -p com.intellij.openapi.application.ReadAction) || exit 1; printf '%s\n' "$api" | grep -q ' compute(' || exit 1; printf '%s\n' "$api" | grep -q 'computeBlocking' && exit 1; grep -q 'T computeBlocking(' "$IJ_SRC/platform/core-api/src/com/intellij/openapi/application/ReadAction.java"
---

## Read or write the code model safely

PSI, VFS and the project model sit behind a read/write lock: many concurrent readers,
or one writer with nobody else reading. Reading without a read action, or writing off
the EDT, is a platform contract violation, not a style choice.

**Wrong (touch the model with no read action):**

```kotlin
fun outsideAnyReadAction(element: PsiElement) {
    element.reference?.resolve() // asserts or throws depending on context
}
```

**Right — blocking, only from the EDT or under modal progress:**

```kotlin
ReadAction.compute<PsiElement?, Throwable> { element.reference?.resolve() }
WriteAction.run<Throwable> { element.delete() } // write actions run on the EDT
```

**Right — non-blocking, from a background thread (preferred there):**

```kotlin
ReadAction.nonBlocking<PsiElement?> { element.reference?.resolve() }
    .expireWith(disposable)
    .finishOnUiThread(ModalityState.defaultModalityState()) { /* use result */ }
    .submit(AppExecutorUtil.getAppExecutorService())
```

Which blocking form is current depends on the build you compile against, and the
platform renamed it mid-stream — read your own target before choosing, see
[source-lookup-target-build.md](source-lookup-target-build.md). At the pinned baseline
`intellijIdea("2025.2.6.2")` (the resolved distribution's own `build.txt` reads
`IU-252.28539.54`), `javap -classpath <dist>/lib/util-8.jar -p
com.intellij.openapi.application.ReadAction` lists only `run`, `compute`,
`nonBlocking(Runnable)`, `nonBlocking(Callable)` and `computeCancellable`:
**`computeBlocking`/`runBlocking` do not exist there**, and `javap -v` puts a
`Deprecated` attribute on neither the static `compute` nor the static `run`. On a 262
build both new names exist and `compute`/`run` do carry `Deprecated: true` — that is
where the rename applies, and where `computeBlocking`/`runBlocking` are what to write.
`ReadAction.nonBlocking(Callable)` is present and undeprecated in both and stays the
background-safe entry point either way. `WriteAction.run`/`compute` are deprecated in
neither build but must still run on the EDT; `WriteAction.runAndWait` transfers control
there from any thread and blocks until it finishes.

Coroutine code uses the suspend equivalents instead of the blocking calls above:
`readAction { }`, `writeAction { }` and `smartReadAction(project) { }` — see
[threading-coroutines.md](threading-coroutines.md).

Holding a `PsiElement` across two separate read actions risks AP-05; cache a
`SmartPsiElementPointer` (`SmartPointerManager.createPointer(element)`) instead — it
survives document changes and returns `null` rather than throwing once invalid.

Reference: `platform/core-api/src/com/intellij/openapi/application/ReadAction.java`;
`platform/core-api/src/com/intellij/openapi/application/WriteAction.java`;
`platform/core-api/src/com/intellij/openapi/application/coroutines.kt`;
`platform/util/src/com/intellij/util/concurrency/AppExecutorUtil.java`
(`getAppExecutorService`); `platform/core-api/src/com/intellij/psi/SmartPointerManager.java`
(`createPointer`).

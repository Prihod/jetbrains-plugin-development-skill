---
title: Decide which thread to run on
tags: threading, edt, background, modality, progress, cancellation
verify: IJ_SRC="${IJ_SRC:?}"; rf=references/threading-model.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$rf"); printf '%s\n' "$body" | grep -qF 'RequiresEdt' || exit 1; printf '%s\n' "$body" | grep -qF 'RequiresBackgroundThread' || exit 1; printf '%s\n' "$body" | grep -qF 'RequiresReadLock' || exit 1; printf '%s\n' "$body" | grep -qF 'RequiresWriteLock' || exit 1; printf '%s\n' "$body" | grep -qF 'assertSlowOperationsAreAllowed()' || exit 1; printf '%s\n' "$body" | grep -qF 'nonModal()' || exit 1; printf '%s\n' "$body" | grep -qF 'stateForComponent(' || exit 1; printf '%s\n' "$body" | grep -qF 'defaultModalityState()' || exit 1; printf '%s\n' "$body" | grep -qF 'checkCanceled()' || exit 1; printf '%s\n' "$body" | grep -qF 'Backgroundable' || exit 1; for f in RequiresEdt RequiresBackgroundThread RequiresReadLock RequiresWriteLock; do test -f "$IJ_SRC/platform/core-api/src/com/intellij/util/concurrency/annotations/$f.java" || exit 1; done && grep -q "public static void assertSlowOperationsAreAllowed()" "$IJ_SRC/platform/core-api/src/com/intellij/util/SlowOperations.java" && grep -q "public static @NotNull ModalityState nonModal()" "$IJ_SRC/platform/core-api/src/com/intellij/openapi/application/ModalityState.java" && grep -q "public static @NotNull ModalityState stateForComponent(" "$IJ_SRC/platform/core-api/src/com/intellij/openapi/application/ModalityState.java" && grep -q "public static @NotNull ModalityState defaultModalityState()" "$IJ_SRC/platform/core-api/src/com/intellij/openapi/application/ModalityState.java" && grep -q "public static void checkCanceled()" "$IJ_SRC/platform/core-api/src/com/intellij/openapi/progress/ProgressManager.java" && grep -q "class Backgroundable extends" "$IJ_SRC/platform/core-api/src/com/intellij/openapi/progress/Task.java"
---

## Decide which thread to run on

Every non-trivial method answers one question before it is written: does it run on the
Event Dispatch Thread (UI, fast, must never block) or on a background thread (may
block, must never touch Swing)? The answer also has a version — verify it for the
target platform release; do not carry forward an old `invokeLater`, `ReadAction`/
`WriteAction`, or coroutine example mechanically. Threading APIs are exactly where
recalled knowledge goes stale.

**Wrong:**

```kotlin
override fun update(e: AnActionEvent) {
    Thread.sleep(50) // any blocking call reached from EDT freezes the UI — AP-04
}
```

**Right (declare the contract instead of guessing it):**

```kotlin
@RequiresBackgroundThread
fun computeSlowly(): Result { /* I/O, indexing, heavy PSI work */ }

@RequiresEdt
fun renderResult(result: Result) { /* Swing only, no PSI resolve — AP-03 */ }
```

`@RequiresEdt`, `@RequiresBackgroundThread`, `@RequiresReadLock`, `@RequiresWriteLock`
live in `platform/core-api/src/com/intellij/util/concurrency/annotations/`. They are
not documentation-only: the DevKit plugin instruments annotated members with
`ThreadingAssertions` calls, so a violated contract throws at runtime instead of
silently corrupting state.

## Modality

Background work that resumes on the EDT does so under a `ModalityState`, which decides
which modal dialogs may still run code meanwhile. `ModalityState.nonModal()` (no modal
dialogs open), `.stateForComponent(component)`, and `.defaultModalityState()` are the
entry points; picking the wrong one lets a stale background result apply while an
unrelated dialog is open, or drops it while nothing is listening.

Reference: `platform/core-api/src/com/intellij/openapi/application/ModalityState.java`.

## Progress and cancellation

Long-running work carries a `ProgressIndicator` — via `Task.Backgroundable` or
`ReadAction.nonBlocking` (see [threading-read-write.md](threading-read-write.md)) — and
checks it periodically with `ProgressManager.checkCanceled()`. A canceled indicator
makes that call throw `ProcessCanceledException`; the exception propagates, it is never
caught and swallowed.

Reference: `platform/core-api/src/com/intellij/openapi/progress/ProgressManager.java`;
`platform/core-api/src/com/intellij/openapi/progress/Task.java`.

## `SlowOperations`

`SlowOperations.assertSlowOperationsAreAllowed()` backs a runtime assertion that some
call paths use to catch slow work reached from the EDT. Coverage is partial — its
silence is not proof of correctness (AP-04).

Reference: `platform/core-api/src/com/intellij/util/SlowOperations.java`.

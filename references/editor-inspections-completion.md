---
title: Add an inspection, intention, completion contributor, or inlay hint
tags: inspections, intentions, completion, inlay-hints, code-insight
verify: IJ_SRC="${IJ_SRC:?}"; f=references/editor-inspections-completion.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'InspectionProfileEntry' || exit 1; printf '%s\n' "$body" | grep -qF 'FileModifier' || exit 1; printf '%s\n' "$body" | grep -qF 'CompletionContributor.extend(' || exit 1; printf '%s\n' "$body" | grep -qF 'InlayHintsProvider.createCollector(' || exit 1; grep -q "public abstract class LocalInspectionTool extends InspectionProfileEntry" "$IJ_SRC/platform/analysis-api/src/com/intellij/codeInspection/LocalInspectionTool.java" && grep -q "public interface IntentionAction extends FileModifier" "$IJ_SRC/platform/analysis-api/src/com/intellij/codeInsight/intention/IntentionAction.java" && grep -q "public abstract class CompletionContributor implements PossiblyDumbAware" "$IJ_SRC/platform/analysis-api/src/com/intellij/codeInsight/completion/CompletionContributor.java" && grep -q "interface InlayHintsProvider : PossiblyDumbAware" "$IJ_SRC/platform/lang-api/src/com/intellij/codeInsight/hints/declarative/InlayHintsProvider.kt" && grep -q "public final void extend(" "$IJ_SRC/platform/analysis-api/src/com/intellij/codeInsight/completion/CompletionContributor.java" && grep -q "fun createCollector(file: PsiFile, editor: Editor)" "$IJ_SRC/platform/lang-api/src/com/intellij/codeInsight/hints/declarative/InlayHintsProvider.kt"
---

## Add an inspection, intention, completion contributor, or inlay hint

Four extension points hook into the editor's code-insight pipeline. Each analyzes PSI
under a read action (see [model-psi.md](model-psi.md)); only an actual document edit
needs a write action — getting that boundary wrong is the trap common to all four.

**Inspections.** `LocalInspectionTool`
(`platform/analysis-api/.../codeInspection/LocalInspectionTool.java:37`, extends
`InspectionProfileEntry`) reports problems via `ProblemsHolder.registerProblem(...)`
(`.../codeInspection/ProblemsHolder.java:81`) from a `PsiElementVisitor` returned by
`buildVisitor()` — it never edits PSI itself. Registered as `localInspection`; worked in
`comparing_string_references_inspection` and `code_inspection_qodana` (mapped in
[source-lookup-samples.md](source-lookup-samples.md)).

**Intentions.** `IntentionAction`
(`platform/analysis-api/.../intention/IntentionAction.java:35`, extends `FileModifier`)
runs `invoke()` inside a command, and inside a write action too if
`startInWriteAction()` returns `true` — the default in
`BaseIntentionAction`/`PsiElementBaseIntentionAction`
(`platform/analysis-api/.../intention/impl/BaseIntentionAction.java:27`). Anything slow
or UI-blocking (a dialog) has no business running while that write action is held:

**Wrong (dialog shown while the platform holds the write action for you):**

```kotlin
class MyIntention : PsiElementBaseIntentionAction() {
    // startInWriteAction() defaults to true — invoke() already runs inside a write action
    override fun invoke(project: Project, editor: Editor?, element: PsiElement) {
        val value = Messages.showInputDialog(project, "Value?", "Title", null) ?: return
        element.replace(makeReplacement(value)) // ok, but the dialog just blocked under a write lock
    }
}
```

**Right — opt out, then take the write action only around the edit:**

```kotlin
class MyIntention : PsiElementBaseIntentionAction() {
    override fun startInWriteAction() = false

    override fun invoke(project: Project, editor: Editor?, element: PsiElement) {
        val value = Messages.showInputDialog(project, "Value?", "Title", null) ?: return
        WriteCommandAction.runWriteCommandAction(project) { element.replace(makeReplacement(value)) }
    }
}
```

Registered as `intentionAction`; worked in `conditional_operator_intention` (mapped in
[source-lookup-samples.md](source-lookup-samples.md)).

**Completion.** `CompletionContributor.extend(type, pattern, provider)`
(`.../completion/CompletionContributor.java:145`) is called from the contributor's own
constructor, not lazily; `provider` is a `CompletionProvider`
(`.../completion/CompletionProvider.java:18`, `addCompletions(...)`).
`CompletionContributor`'s own javadoc documents this whole pipeline as running inside a
read action (`.../CompletionContributor.java:159-161`) — a blocking call there freezes
typing since it can't honor `ProgressManager.checkCanceled()`. Registered as
`completion.contributor`.

**Inlay hints.** `InlayHintsProvider.createCollector(file, editor)`
(`platform/lang-api/.../hints/declarative/InlayHintsProvider.kt:30`) returns a collector
that reads PSI/document state to place hints; it never writes. Registered as
`codeInsight.declarativeInlayProvider`.

Reference: `platform/analysis-api/src/com/intellij/codeInspection/LocalInspectionTool.java`;
`.../codeInspection/ProblemsHolder.java`; `platform/core-api/src/com/intellij/psi/PsiElementVisitor.java`;
`.../codeInsight/intention/IntentionAction.java`; `.../codeInsight/intention/impl/BaseIntentionAction.java`;
`platform/lang-api/src/com/intellij/codeInsight/intention/PsiElementBaseIntentionAction.java`;
`.../codeInsight/completion/CompletionContributor.java`; `.../codeInsight/completion/CompletionProvider.java`;
`platform/lang-api/src/com/intellij/codeInsight/hints/declarative/InlayHintsProvider.kt`.
EP declarations: `platform/analysis-api/resources/META-INF/Analysis.analyzer.xml`
(`localInspection`, `intentionAction`, as `qualifiedName`); `.../Analysis.xml`
(`completion.contributor`); `platform/platform-resources/src/META-INF/LangExtensionPoints.xml`
(`codeInsight.declarativeInlayProvider`).

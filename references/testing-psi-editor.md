---
title: Commit the Document before reading PSI in a test
tags: testing, psi, editor, fixtures
verify: IJ_SRC="${IJ_SRC:?}"; PLUGIN_TEMPLATE_HOME="${PLUGIN_TEMPLATE_HOME:?}"; f=references/testing-psi-editor.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'commitDocument' || exit 1; printf '%s\n' "$body" | grep -qF 'PsiErrorElementUtil.hasErrors(project, xmlFile.virtualFile)' || exit 1; grep -n "public abstract void commitDocument(@NotNull Document document);" "$IJ_SRC/platform/core-api/src/com/intellij/psi/PsiDocumentManager.java" && grep -n "PsiErrorElementUtil.hasErrors(project, xmlFile.virtualFile)" "$PLUGIN_TEMPLATE_HOME/src/test/kotlin/org/jetbrains/plugins/template/MyPluginTest.kt"
---

## Commit the Document before reading PSI in a test

An editor's `Document` and its PSI tree can be out of sync until the edit is committed;
a test that edits the `Document` directly and reads PSI right after can observe a stale
or partially-reparsed tree.

**Wrong (raw Document edit, no commit):**

```kotlin
val psiFile = myFixture.configureByText("a.txt", "foo")
val document = myFixture.getDocument(psiFile)
document.insertString(3, "bar")
assertEquals("foobar", psiFile.text) // may still see the pre-edit tree here
```

**Right (commit explicitly, or let the fixture do it):**

```kotlin
val psiFile = myFixture.configureByText("a.txt", "foo")
val document = myFixture.getDocument(psiFile)
document.insertString(3, "bar")
PsiDocumentManager.getInstance(project).commitDocument(document)
assertEquals("foobar", psiFile.text)

// or avoid the raw edit — myFixture.type() commits for you:
myFixture.type("bar")
```

`myFixture.getDocument(psiFile)` above is `CodeInsightTestFixture.getDocument(PsiFile)`
(`CodeInsightTestFixture.java:669`) — a different method from
`PsiDocumentManager.getDocument(PsiFile)` (see [model-psi.md](model-psi.md)), despite the
identical name.

`PsiDocumentManager.commitDocument(Document)`
(`platform/core-api/src/com/intellij/psi/PsiDocumentManager.java:128`, class at `:23`) is
the explicit sync point. `CodeInsightTestFixture.type(String)`
(`platform/testFramework/src/com/intellij/testFramework/fixtures/CodeInsightTestFixture.java:685`)
and `configureByText(String, String)` (`:196`) drive the real typing/parsing path and
keep Document and PSI in sync without a manual call.

## Comparing results and asserting on PSI

For "run an action, check the outcome" tests, use `checkResultByFile(String)`
(`CodeInsightTestFixture.java:238`) against a `_after` file in `testData` rather than
asserting on fragments of `psiFile.text` — see
[testing-levels-fixtures.md](testing-levels-fixtures.md) for the `foo.xml`/`foo_after.xml`
convention. `completeBasic()` (`:648`) drives completion tests the same way: configure,
invoke, assert on the returned `LookupElement[]`
(`LookupElement`, `platform/analysis-api/src/com/intellij/codeInsight/lookup/LookupElement.java:27`).

For structural PSI assertions, walk the tree instead of matching substrings in
`psiFile.text` — `PsiErrorElementUtil.hasErrors(Project, VirtualFile)`
(`platform/platform-impl/src/com/intellij/util/PsiErrorElementUtil.java:26`) is the
template's own check that a parsed file has no `PsiErrorElement`s
(`platform/core-api/src/com/intellij/psi/PsiErrorElement.java:15`, interface extends
`PsiElement`), used exactly as
`PsiErrorElementUtil.hasErrors(project, xmlFile.virtualFile)` in `MyPluginTest.kt`.

Reference: `platform/testFramework/src/com/intellij/testFramework/fixtures/CodeInsightTestFixture.java`;
`platform/core-api/src/com/intellij/psi/PsiDocumentManager.java`;
`platform/platform-impl/src/com/intellij/util/PsiErrorElementUtil.java`;
`platform/analysis-api/src/com/intellij/codeInsight/lookup/LookupElement.java`;
`platform/core-api/src/com/intellij/psi/PsiErrorElement.java`;
`$PLUGIN_TEMPLATE_HOME/src/test/kotlin/org/jetbrains/plugins/template/MyPluginTest.kt`.

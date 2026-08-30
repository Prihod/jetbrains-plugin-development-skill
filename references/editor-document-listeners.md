---
title: Read editor state under a read action, change the document under a write action
tags: editor, document, caret, selection, listeners
verify: IJ_SRC="${IJ_SRC:?}"; f=references/editor-document-listeners.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); n=$(printf '%s\n' "$body" | grep -cF 'UserDataHolder'); [ "$n" -ge 2 ] || exit 1; grep -n "public interface Document extends UserDataHolder" "$IJ_SRC/platform/core-api/src/com/intellij/openapi/editor/Document.java" && grep -n "public interface Editor extends UserDataHolder" "$IJ_SRC/platform/editor-ui-api/src/com/intellij/openapi/editor/Editor.java"
---

## Read editor state under a read action, change the document under a write action

`Editor` (`platform/editor-ui-api/.../Editor.java`, interface extending `UserDataHolder`)
exposes a `Document` (`platform/core-api/.../Document.java`, also extends
`UserDataHolder`) via `getDocument()`, plus a `CaretModel` and `SelectionModel` via
`getCaretModel()` / `getSelectionModel()`. `Document` is the mutable-text-with-offsets
layer below PSI — see [model-psi.md](model-psi.md) for how the two relate.

The through-rule: reading this state — text, caret position, selection, or PSI built on
top of it — happens under a read action; changing the document happens under a write
action, and a write action only runs on the EDT. The full read/write-action contract is
in [threading-model.md](threading-model.md) and
[threading-read-write.md](threading-read-write.md); this file only applies it here.

**Wrong (mutate the document with no write action):**

```kotlin
fun replaceSelection(editor: Editor, text: String) {
    val sel = editor.selectionModel
    editor.document.replaceString(sel.selectionStart, sel.selectionEnd, text)
    // fails: no write action in progress (Application.assertWriteAccessAllowed)
}
```

**Right — wrap the mutation in a write command on the EDT:**

```kotlin
fun replaceSelection(project: Project, editor: Editor, text: String) {
    val sel = editor.selectionModel
    WriteCommandAction.runWriteCommandAction(project) {
        editor.document.replaceString(sel.selectionStart, sel.selectionEnd, text)
    }
}
```

`WriteCommandAction` wraps the write action in an undoable command; `insertString`,
`deleteString` and `replaceString`
(`platform/core-api/.../Document.java`) all carry this requirement. Worked
end-to-end in `editor_basics` (mapped in
[source-lookup-samples.md](source-lookup-samples.md)):
`EditorIllustrationAction.java` calls
`WriteCommandAction.runWriteCommandAction(project, () -> document.replaceString(...))`.

## Listening for changes

`Document.addDocumentListener(listener, parentDisposable)`
(`platform/core-api/.../Document.java`) fires `DocumentListener.documentChanged`
(`platform/core-api/.../event/DocumentListener.java`) for one document.
`CaretModel.addCaretListener` / `SelectionModel.addSelectionListener`
(`platform/editor-ui-api/.../CaretModel.java`,
`.../SelectionModel.java`) do the same for one editor's caret and selection. To
listen across every open editor instead of one document,
`EditorFactory.getInstance().getEventMulticaster()`
(`platform/editor-ui-api/.../EditorFactory.java`) exposes the identical
`addDocumentListener` / `addCaretListener` / `addSelectionListener` methods globally.

All three have a two-argument overload taking a `Disposable` parent — use it. The
single-argument `Document.addDocumentListener(listener)` is `@deprecated` outright; the
caret/selection single-argument forms merely leave you to call
`removeCaretListener`/`removeSelectionListener` yourself. Skip that and the listener
outlives whatever installed it — the same leak
[extend-listeners-dynamic.md](extend-listeners-dynamic.md) covers for the message bus
(AP-06).

Reference: `platform/core-api/src/com/intellij/openapi/editor/Document.java`;
`platform/core-api/src/com/intellij/openapi/command/WriteCommandAction.java`;
`platform/editor-ui-api/src/com/intellij/openapi/editor/Editor.java`;
`platform/editor-ui-api/src/com/intellij/openapi/editor/CaretModel.java`;
`platform/editor-ui-api/src/com/intellij/openapi/editor/SelectionModel.java`;
`platform/core-api/src/com/intellij/openapi/editor/event/DocumentListener.java`;
`platform/editor-ui-api/src/com/intellij/openapi/editor/EditorFactory.java`;
`platform/editor-ui-api/src/com/intellij/openapi/editor/event/EditorEventMulticaster.java`;
`intellij-sdk-code-samples/editor_basics/src/main/java/org/intellij/sdk/editor/EditorIllustrationAction.java`.

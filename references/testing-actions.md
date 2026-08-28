---
title: Test an AnAction with a DataContext you built yourself
tags: testing, actions, data-context, fixtures
verify: IJ_SRC="${IJ_SRC:?}"; f=references/testing-actions.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); TF="$IJ_SRC/platform/testFramework/src/com/intellij/testFramework"; grep -q 'AnActionEvent createTestEvent(@NotNull AnAction action)' "$TF/TestActionEvent.java" || exit 1; grep -q 'context != null ? context : DataManager.getInstance().getDataContext()' "$TF/TestActionEvent.java" || exit 1; grep -q 'AnActionEvent e = TestActionEvent.createTestEvent(action);' "$TF/fixtures/impl/CodeInsightTestFixtureImpl.java" || exit 1; grep -q 'if (e.getPresentation().isEnabled())' "$TF/fixtures/impl/CodeInsightTestFixtureImpl.java" || exit 1; grep -q 'setDataProvider(new TestDataProvider(getProject()))' "$TF/fixtures/impl/LightIdeaTestFixtureImpl.java" || exit 1; grep -q 'CommonDataKeys.EDITOR.is(dataId)' "$TF/TestDataProvider.java" || exit 1; SD="$IJ_SRC/platform/ide-core-impl/src/com/intellij/openapi/actionSystem/impl/SimpleDataContext.java"; grep -q 'public static @NotNull Builder builder()' "$SD" || exit 1; grep -q 'public @NotNull DataContext build()' "$SD" || exit 1; for s in 'SimpleDataContext.builder()' 'TestActionEvent.createTestEvent(action, ctx)' 'TestDataProvider' 'ProjectViewPopupMenu' 'PROJECT=set VIRTUAL_FILE=null'; do printf '%s\n' "$body" | grep -qF -- "$s" || exit 1; done
---

## Test an `AnAction` with a `DataContext` you built yourself

`myFixture.testAction(action)` is the obvious way to test an action and it is not a
neutral harness: it decides the action's `DataContext` for you, and what it decides is
"whatever the fixture's open editor implies".

`CodeInsightTestFixtureImpl.testAction` (`:1138-1145`) calls
`TestActionEvent.createTestEvent(action)` with no `DataContext`; that overload falls back
to `DataManager.getInstance().getDataContext()` (`TestActionEvent.java:62-69`). Under a
light fixture the data provider behind it is `TestDataProvider`
(installed at `LightIdeaTestFixtureImpl.java:59`), which serves `PROJECT`, `EDITOR`,
`NAVIGATE_IN_EDITOR` and `FILE_EDITOR` (`TestDataProvider.java:42-58`) — everything else
an action reads is derived from that editor by the platform's data rules.

**Measured on this Baseline** with a probe action that reads each key inside `update()`
(`set` = non-null):

```
testAction, after configureByText("probe.txt"):
  PROJECT=set VIRTUAL_FILE=probe.txt VIRTUAL_FILE_ARRAY=1 PSI_FILE=probe.txt EDITOR=set
  PSI_ELEMENT=null NAVIGATABLE=null SELECTED_ITEMS=null
testAction, with nothing configured:
  PROJECT=set VIRTUAL_FILE=null VIRTUAL_FILE_ARRAY=null PSI_FILE=null EDITOR=null
```

So the trap is not that the context is empty — it is that the context is always the
**editor's**. An action registered in `ProjectViewPopupMenu`, whose real context is a tree
selection (`SELECTED_ITEMS`, `NAVIGATABLE_ARRAY`, `PSI_ELEMENT` — see
[extend-actions.md](extend-actions.md)), is never exercised by `testAction`: the keys it
reads are null, `update()` disables the action, `testAction` therefore skips
`actionPerformed()` entirely, and the test still passes. Forget `configureByText` and even
an editor action tests nothing.

**Right — state the context the action is registered for:**

```kotlin
val file = myFixture.configureByText("probe.txt", "hello")
val action = ShowFileInfoAction()
val ctx = SimpleDataContext.builder()
    .add(CommonDataKeys.PROJECT, project)
    .add(CommonDataKeys.VIRTUAL_FILE, file.virtualFile)
    .build()
val e = TestActionEvent.createTestEvent(action, ctx)
action.update(e)
assertTrue(e.presentation.isEnabled)   // assert the enablement you meant to test
```

`SimpleDataContext.builder()` (`:64`) → `add(DataKey, T)` → `build()` (`:108`) is the
type-safe form; `getSimpleContext(Map, DataContext)` is deprecated in the same file.
`TestActionEvent.createTestEvent(AnAction, DataContext)` (`:57`) is the overload that
takes the context — the single-argument one is the one that does not.

Assert on the `Presentation` (`e.presentation.isEnabled` / `isVisible`) for `update()`
tests, and drive `actionPerformed(e)` explicitly for the behaviour test, so a disabled
presentation fails the test instead of silently skipping the work.

Reference: `platform/testFramework/src/com/intellij/testFramework/TestActionEvent.java`;
`platform/testFramework/src/com/intellij/testFramework/TestDataProvider.java`;
`platform/testFramework/src/com/intellij/testFramework/fixtures/impl/CodeInsightTestFixtureImpl.java`
and `LightIdeaTestFixtureImpl.java`;
`platform/ide-core-impl/src/com/intellij/openapi/actionSystem/impl/SimpleDataContext.java`;
[testing-levels-fixtures.md](testing-levels-fixtures.md).

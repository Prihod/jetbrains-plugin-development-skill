---
title: Pick the lowest test level that proves the behavior
tags: testing, fixtures, test-framework, testdata
verify: IJ_SRC="${IJ_SRC:?}"; PLUGIN_TEMPLATE_HOME="${PLUGIN_TEMPLATE_HOME:?}"; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' references/testing-levels-fixtures.md); norm=$(printf '%s' "$body" | tr '\n' ' ' | tr -s ' '); tgt=references/lifecycle-disposable-messagebus.md; printf '%s' "$norm" | grep -qF 'worked through as a `HeavyPlatformTestCase` in [lifecycle-disposable-messagebus.md](lifecycle-disposable-messagebus.md)' || exit 1; grep -qF ' : HeavyPlatformTestCase() {' "$tgt" || exit 1; grep -qF 'PlatformTestUtil.forceCloseProjectWithoutSaving(project)' "$tgt" || exit 1; grep -n "public abstract class BasePlatformTestCase extends UsefulTestCase {" "$IJ_SRC/platform/testFramework/src/com/intellij/testFramework/fixtures/BasePlatformTestCase.java" && grep -n "testFramework(TestFrameworkType.Platform)" "$PLUGIN_TEMPLATE_HOME/build.gradle.kts" && find "$PLUGIN_TEMPLATE_HOME/src/test/testData/rename" -name "foo_after.xml" | grep -q .
---

## Pick the lowest test level that proves the behavior

The platform test framework spans five levels of increasing cost. Reaching for a
heavier level than a change needs slows the suite without adding verification.

```
Unit -> Light -> Integration -> IDE tests -> UI tests
```

- **Unit** — plain JUnit, no platform bootstrap.
- **Light** — `BasePlatformTestCase` (`platform/testFramework/src/com/intellij/testFramework/fixtures/BasePlatformTestCase.java:29`, extends `UsefulTestCase`) or `LightJavaCodeInsightFixtureTestCase` (`java/testFramework/src/com/intellij/testFramework/fixtures/LightJavaCodeInsightFixtureTestCase.java:68`): an in-memory project, exposed as `myFixture` (`CodeInsightTestFixture`, `platform/testFramework/src/com/intellij/testFramework/fixtures/CodeInsightTestFixture.java:82`).
- **Integration** — `HeavyPlatformTestCase` (`platform/testFramework/src/com/intellij/testFramework/HeavyPlatformTestCase.java:98`): a real project on disk; its own javadoc recommends light tests "whenever possible" for the performance difference.
- **IDE tests** — IntelliJ IDE Starter's `Starter.newContext()` (`tools/intellij.tools.ide.starter/src/com/intellij/ide/starter/runner/Starter.kt:9`, method at `:21`) launches a real, installed IDE process.
- **UI tests** — the `Driver` interface (`platform/remote-driver/client/src/com/intellij/driver/client/Driver.kt:65`) drives that running IDE's actual UI.

Neither `$IJ_SAMPLES` nor the template has a worked example for the IDE tests or UI tests rungs; the two citations above point at the class declarations only.

**Wrong (Integration level for a claim Light already covers):**

```kotlin
class MyPluginTest : HeavyPlatformTestCase() {
    fun testXmlIsParsed() {
        // spins up a real on-disk project just to parse one XML snippet
    }
}
```

**Right (Light fixture — same coverage, in-memory):**

```kotlin
class MyPluginTest : BasePlatformTestCase() {
    fun testXmlIsParsed() {
        myFixture.configureByText(XmlFileType.INSTANCE, "<foo>bar</foo>")
    }
}
```

`configureByText(FileType, String)` (`CodeInsightTestFixture.java:186`) and `XmlFileType`
(`xml/xml-psi-impl/src/com/intellij/ide/highlighter/XmlFileType.java:12`) are the
identifiers above, taken from the template's `MyPluginTest.kt`.

## Wiring and layout

The framework artifact is declared once, in the build —
`testFramework(TestFrameworkType.Platform)` inside `dependencies { intellijPlatform { } }`;
see [setup-build.md](setup-build.md) for the full block. `$PLUGIN_TEMPLATE_HOME/build.gradle.kts:15`
carries it verbatim.

Fixture-backed tests read from `src/test/testData`, one subdirectory per feature;
`getTestDataPath()` (`BasePlatformTestCase.java:98`) points at it, and `@TestDataPath`
(`platform/testFramework/src/com/intellij/testFramework/TestDataPath.java:20`) documents
the same path for IDE navigation (it does not affect test execution). A
before/after pair — `foo.xml` and `foo_after.xml` under `src/test/testData/rename/` in the
template — is the standard shape for "run an action, compare the result":
`myFixture.testRename("foo.xml", "foo_after.xml", "a2")`
(`CodeInsightTestFixture.java:543`, the 3-arg-plus-varargs overload).

A bug fix ships with a regression test at the lowest level that reproduces it — most PSI/editor fixes
reproduce at Light; reserve Integration and above for behavior only a heavier fixture can observe.
Disposal on project close is that case (Light never closes its shared project), worked through as a
`HeavyPlatformTestCase` in [lifecycle-disposable-messagebus.md](lifecycle-disposable-messagebus.md).

Testing an `AnAction` is its own trap — `myFixture.testAction` picks the data context for you: [testing-actions.md](testing-actions.md).

Reference: `platform/testFramework/src/com/intellij/testFramework/fixtures/BasePlatformTestCase.java`,
`CodeInsightTestFixture.java`;
`java/testFramework/src/com/intellij/testFramework/fixtures/LightJavaCodeInsightFixtureTestCase.java`;
`platform/testFramework/src/com/intellij/testFramework/HeavyPlatformTestCase.java`,
`TestDataPath.java`;
`tools/intellij.tools.ide.starter/src/com/intellij/ide/starter/runner/Starter.kt`;
`platform/remote-driver/client/src/com/intellij/driver/client/Driver.kt`;
`xml/xml-psi-impl/src/com/intellij/ide/highlighter/XmlFileType.java`;
`$PLUGIN_TEMPLATE_HOME/build.gradle.kts`, `src/test/kotlin/org/jetbrains/plugins/template/MyPluginTest.kt`,
`src/test/testData/rename/`.

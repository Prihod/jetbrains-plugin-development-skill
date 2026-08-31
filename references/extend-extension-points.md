---
title: Find or declare an extension point
tags: extension-points, plugin-xml, extensibility
verify: IJ_SRC="${IJ_SRC:?}"; f=references/extend-extension-points.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'com.intellij.codeInsight.containerProvider' || exit 1; printf '%s\n' "$body" | grep -qF 'dynamic="true"' || exit 1; grep -n -A2 'qualifiedName="com.intellij.codeInsight.containerProvider"' "$IJ_SRC/platform/analysis-api/resources/META-INF/Analysis.analyzer.xml" | grep -q 'dynamic="true"'
---

## Find or declare an extension point

Before writing a new pluggable-behavior mechanism, check whether the platform already
has an extension point (EP) for it — see AP-13. Reinventing one compiles fine and still
duplicates a mechanism every other plugin already knows how to use.

**Search first:**

```bash
# ripgrep; `grep -rn --include='*.xml'` does the same without it.
rg -n '<extensionPoint[^>]*name="<EP-NAME>"' "$IJ_SRC" -g '*.xml'
```

See [source-lookup.md#search-recipes](source-lookup.md#search-recipes) for the full
recipe set, including how to find who already registers against a candidate EP.

**Wrong (reinvent instead of search — AP-13):**

```kotlin
object MyRegistry { val handlers = mutableListOf<Handler>() } // reinvents an EP
```

**Right (declare the EP; `dynamic="true"` so registrants load/unload without a restart
— see AP-08):**

```xml
<extensionPoints>
    <extensionPoint qualifiedName="com.intellij.codeInsight.containerProvider"
                     interface="com.intellij.codeInsight.ContainerProvider"
                     dynamic="true"/>
</extensionPoints>
```

```java
public interface ContainerProvider {
  ExtensionPointName<ContainerProvider> EP_NAME =
      ExtensionPointName.create("com.intellij.codeInsight.containerProvider");

  @Nullable PsiElement getContainer(@NotNull PsiElement item);
}
```

Consume it with `EP_NAME.getExtensionList()` (or `forEachExtensionSafe`); a plugin
registers an implementation with `<extensions defaultExtensionNs="...">`, matching the
`qualifiedName` above. Omitting `dynamic="true"` still compiles — see AP-08.

Reference: `platform/core-api/src/com/intellij/codeInsight/ContainerProvider.java`;
`platform/analysis-api/resources/META-INF/Analysis.analyzer.xml`.

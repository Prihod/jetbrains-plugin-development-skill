---
title: Check an API against the target build, not against the sources
tags: sources, verification, versions, deprecation, baseline
verify: IJ_SRC="${IJ_SRC:?}"; HOME="${HOME:?}"; f=references/source-lookup-target-build.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); srcv=$(cat "$IJ_SRC/build.txt"); d=$(ls -d "$HOME"/.gradle/caches/*/transforms/*/transformed/ideaIU-2025.2.6.2 2>/dev/null | head -1); test -n "$d" || exit 1; bt=$(find "$d" -maxdepth 2 -name build.txt | head -1); test -n "$bt" || exit 1; tgtv=$(cat "$bt"); t=${tgtv##*-}; test "${srcv%%.*}" != "${t%%.*}" || exit 1; printf '%s\n' "$body" | grep -qF "$srcv" || exit 1; printf '%s\n' "$body" | grep -qF "$tgtv" || exit 1; for x in $(printf '%s\n' "$body" | grep -oE '[A-Z]{2}-[0-9]+\.[0-9]+\.[0-9]+'); do test "$x" = "$tgtv" || exit 1; done; for x in $(printf '%s\n' "$body" | grep -oE '[0-9]+\.[0-9]+\.SNAPSHOT'); do test "$x" = "$srcv" || exit 1; done; javap -classpath "$d/lib/util-8.jar" -p com.intellij.openapi.application.ReadAction | grep -q computeBlocking && exit 1; grep -q 'T computeBlocking(' "$IJ_SRC/platform/core-api/src/com/intellij/openapi/application/ReadAction.java"
---

## Check an API against the target build, not against the sources

The sources you read and the build you compile against are two artefacts with two
different version numbers. On this skill's own Baseline they differ by a whole branch:

| Artefact | Its build number is written in | Value on the validated setup |
|---|---|---|
| `intellij-community` checkout (`$IJ_SRC`) | `$IJ_SRC/build.txt` | `262.8665.SNAPSHOT` |
| resolved `intellijIdea("2025.2.6.2")` distribution | `build.txt` inside the extracted distribution under `~/.gradle/caches/<gradle-version>/transforms/*/transformed/ideaIU-<version>/` | `IU-252.28539.54` |

Sources run ahead of any released build. So a symbol present in `$IJ_SRC` is **not**
thereby present at your target, and a `@Deprecated` read in `$IJ_SRC` is **not** thereby
in effect at your target. "X is current, Y is deprecated" is always a claim about one
build; before carrying it across, read both numbers above and see whether they agree.

**This is not hypothetical.** `ReadAction.computeBlocking` is declared in
`$IJ_SRC/platform/core-api/src/com/intellij/openapi/application/ReadAction.java`, where
`compute` is `@Deprecated` in its favour. At build 252 `computeBlocking` does not exist
at all and `compute` carries no deprecation, so code written from the sources fails with
`Unresolved reference: computeBlocking` — see
[threading-read-write.md](threading-read-write.md).

### Checking one symbol at the target build

```bash
# 1. What build is the target? Read it from the distribution, do not infer it.
DIST=$(ls -d ~/.gradle/caches/*/transforms/*/transformed/ideaIU-*/ | head -1)
find "$DIST" -maxdepth 2 -name build.txt -exec cat {} \;   # e.g. IU-252.28539.54

# 2. Which jar carries the class? The layout changes between builds — 252 packs
#    platform classes into lib/util-8.jar, 262 into per-module lib/intellij.platform.*.jar
#    — so search for it instead of guessing the jar name:
for j in "$DIST"/lib/*.jar; do
  unzip -l "$j" 2>/dev/null | grep -q 'com/intellij/openapi/application/ReadAction\.class' && echo "$j"
done

# 3. Does the member exist at that build?
javap -classpath "$JAR" -p com.intellij.openapi.application.ReadAction

# 4. Is it deprecated at that build? Read the bytecode attribute, not the javadoc:
javap -v -classpath "$JAR" com.intellij.openapi.application.ReadAction |
  awk '/^  (public|protected|private|static).*\(/{s=$0} /^    Deprecated: true/{print s}'
```

An installed IDE answers the same questions with `<IDE>/Contents/lib/` as `$DIST/lib/`;
its `build.txt` sits under `<IDE>/Contents/Resources/`. Use whichever artefact the
project actually builds against — if `build.gradle.kts` pins `intellijIdea(...)`, the
installed IDE is not the target and cannot settle the question.

When the two numbers differ and you cannot reach the target's jars, say so and lower
stated confidence, exactly as [source-lookup.md](source-lookup.md) requires — a source
tree is evidence about the sources' branch and nothing else.

Reference: `$IJ_SRC/build.txt`; `build.txt` and `lib/` of the resolved
`ideaIU-<version>` distribution; `SKILL.md`'s Baseline table;
[source-lookup.md](source-lookup.md); [threading-read-write.md](threading-read-write.md).

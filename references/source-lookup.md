---
title: Finding and verifying IntelliJ Platform sources
tags: sources, verification, extension-points, threading, deprecation
verify: IJ_SRC="${IJ_SRC:?}"; IJ_SAMPLES="${IJ_SAMPLES:?}"; PHPSTORM_HOME="${PHPSTORM_HOME:?}"; f=references/source-lookup.md; norm=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f" | tr '\n' ' ' | tr -s ' '); sym=$(printf '%s' "$norm" | grep -oE 'rg -n "interface [A-Za-z0-9]+"' | head -1); sym=${sym#rg -n \"}; sym=${sym%\"}; test -n "$sym" || exit 1; name=${sym#interface }; d=$(find "$IJ_SRC/platform" -name "$name.java" -o -name "$name.kt" | head -1); test -n "$d" || exit 1; grep -q "$sym" "$d" || exit 1; ann=$(printf '%s' "$norm" | grep -oE 'platform/core-api/src/com/intellij/util/concurrency/annotations/' | head -1); test -n "$ann" -a -d "$IJ_SRC/$ann" || exit 1; jar=$(printf '%s' "$norm" | grep -oE 'Contents/plugins/php/lib/[A-Za-z0-9._-]+-src\.jar' | head -1); test -n "$jar" -a -f "$PHPSTORM_HOME/$jar" || exit 1; n=$(printf '%s' "$norm" | sed -E 's/.* ([0-9]+) compilable JetBrains samples.*/\1/'); test "$n" = "$(find "$IJ_SAMPLES" -maxdepth 1 -mindepth 1 -type d ! -name '.*' ! -name '_*' | wc -l | tr -d ' ')"
---

## Finding sources

Never reconstruct an IntelliJ Platform API from memory. Locate a real source and
read it. Search in this order, stopping at the first hit:

1. **Explicit configuration** — an environment variable the project itself sets, or a
   path in project settings. No name is standard; this skill's commands read `IJ_SRC`,
   `IJ_SAMPLES`, `PLUGIN_TEMPLATE_HOME` and `PHPSTORM_HOME`.
2. **An installed JetBrains IDE** — almost always present on a dev machine:
   - macOS: `/Applications/<IDE>.app/Contents/plugins/`, `~/Applications/JetBrains Toolbox/`
   - Linux: `~/.local/share/JetBrains/Toolbox/apps/`
   - Windows: `%LOCALAPPDATA%\Programs\<IDE>\plugins\`
3. **An `intellij-community` checkout** — optional; gives platform implementation,
   tests, and threading annotations.
4. **An `intellij-sdk-code-samples` checkout** — optional; see
   [source-lookup-samples.md](source-lookup-samples.md).
5. **Nothing found** — fall back to the official documentation, say out loud that
   confidence is lower, and never invent class names.

| Source | Uniquely gives | Availability |
|---|---|---|
| Installed IDE | `plugin.xml` of bundled plugins, EP declarations, `*-src.jar` of closed-source plugins | Almost always |
| `intellij-community` | Platform implementation, tests, threading annotations | If cloned |
| `sdk-code-samples` | 21 compilable JetBrains samples | If cloned |
| `plugin-template` | Reference build configuration | If cloned |

**Verified fact:** the PHP plugin is closed-source and is absent from
`intellij-community`. The only authoritative source for PHP API is
`$PHPSTORM_HOME/Contents/plugins/php/lib/php-openapi-src.jar` inside an installed
PhpStorm — no other source substitutes for it.

**Verified fact:** the Gradle cache
(`~/.gradle/caches/modules-2/files-2.1/com.jetbrains.intellij.idea/`) holds platform
artifacts, but `-sources.jar` files may be absent. Do not use it as a source of sources.

## Search recipes

```bash
# These use ripgrep. Without it, `grep -rn` with `--include` does the same job.
# Where is a symbol declared?
rg -n "interface ActionUpdateThreadAware" "$IJ_SRC/platform"

# Is it deprecated, experimental, or internal?
rg -n -B3 "class ServiceManager" "$IJ_SRC/platform/core-api" | rg "Deprecated|ApiStatus"

# What is its threading contract? (annotations live in
# platform/core-api/src/com/intellij/util/concurrency/annotations/)
rg -n "@RequiresEdt|@RequiresReadLock|@RequiresBackgroundThread" <file>

# Who declares this extension point? Some use name="<EP-NAME>" (short, namespaced),
# others qualifiedName="...<EP-NAME>" (already fully qualified — e.g. localInspection,
# intentionAction). Check both; one form alone proves nothing:
rg -n '<extensionPoint[^>]*(name="<EP-NAME>"|qualifiedName="[^"]*\.<EP-NAME>")' "$IJ_SRC" -g '*.xml'

# Who registers against this extension point? (registration tag = EP's short name)
rg -n '<<EP-NAME>[ />]' "$IJ_SRC" -g '*.xml'

# How does the platform test this API?
rg -l "<CLASS-NAME>" "$IJ_SRC" -g '*Test*.kt' -g '*Test*.java'

# Read the API of a closed-source bundled plugin
unzip -l "<IDE>/Contents/plugins/<plugin>/lib/<name>-src.jar"
unzip -p "<IDE>/Contents/plugins/<plugin>/lib/<name>-src.jar" path/To/Class.java

# Which plugin/jar does a class come from, by location? (installed IDE)
for j in "<IDE>/Contents/plugins"/*/lib/*.jar; do
  unzip -l "$j" 2>/dev/null | grep -q "path/To/Class.class" && echo "$j"
done
```

If none of these produce a hit, treat that as a signal to fall back to the official
docs and lower stated confidence — not as license to guess.

Reference: `intellij-community` checkout; installed IDE's `plugins/` directory.

---
title: Depend on the PHP plugin; find PhpStorm API sources
tags: php, phpstorm, dependencies, sources
verify: PHPSTORM_HOME="${PHPSTORM_HOME:?}"; f=references/php-dependency-sources.md; norm=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f" | tr '\n' ' ' | tr -s ' '); D=$(unzip -p "$PHPSTORM_HOME/Contents/plugins/php-impl/lib/php.jar" META-INF/plugin.xml | awk '/<dependencies>/{n++} n==1{print} /<\/dependencies>/{if(n==1) exit}' | grep '<plugin id=' | sed -E 's/.*id="([^"]+)".*/\1/' | sort -u); test -n "$D" || exit 1; printf '%s' "$norm" | grep -qF 'lists these mandatory ids: ' || exit 1; seg=$(printf '%s' "$norm" | sed -E 's/.*lists these mandatory ids: //; s/\. .*//'); B=$(printf '%s\n' "$seg" | grep -oE 'com\.intellij\.modules\.[a-z-]+' | sort -u); test "$D" = "$B" || { printf 'descriptor:\n%s\nfile:\n%s\n' "$D" "$B"; exit 1; }; unzip -p "$PHPSTORM_HOME/Contents/plugins/json/lib/intellij.json.jar" META-INF/plugin.xml | grep -q '<id>com.intellij.modules.json</id>' || exit 1; unzip -p "$PHPSTORM_HOME/Contents/plugins/php-impl/lib/php.jar" META-INF/plugin.xml | grep -q '<id>com.jetbrains.php</id>'
---

## Depend on the PHP plugin; find PhpStorm API sources

`com.jetbrains.php.*` is PhpStorm/PHP-plugin API, not IntelliJ Platform API. Code that
imports it compiles and runs fine while you build against an installed PhpStorm,
because the class sits on that IDE's classpath — and fails to load anywhere the plugin
isn't bundled. Using it requires a declared dependency in **both** the build script and
`plugin.xml`; see [setup-dependency-origin.md](setup-dependency-origin.md) for the
general rule. Getting the build half right and forgetting the `plugin.xml` half is
AP-12 (`antipatterns-dependencies.md`).

**Verified fact:** the PHP plugin's own descriptor —
`$PHPSTORM_HOME/Contents/plugins/php-impl/lib/php.jar!/META-INF/plugin.xml` — declares
`<id>com.jetbrains.php</id>` and requires `<plugin id="com.intellij.modules.ultimate" />`.
It loads only in an Ultimate-based IDE (IntelliJ IDEA Ultimate, PhpStorm, or another IDE
that bundles it), never a Community-based one.

## The PHP plugin's own dependency chain

`bundledPlugin("com.jetbrains.php")` gets it to **compile**. Getting it to **load** also
means satisfying the PHP plugin's own mandatory dependencies — and a missing one is not a
build failure: the PHP plugin refuses to load, your PHP PSI calls return `null`, and a
light-fixture test can still go green on the wrong answer.

Read the chain from the descriptor; it moves between builds:

```bash
PHPSTORM_HOME="${PHPSTORM_HOME:?}"
unzip -p "$PHPSTORM_HOME/Contents/plugins/php-impl/lib/php.jar" META-INF/plugin.xml |
  awk '/<dependencies>/{n++} n==1{print} /<\/dependencies>/{if(n==1) exit}' | grep '<plugin id='
```

The `awk` takes only the **first** `<dependencies>` block — the plugin's own; the later ones
belong to its bundled content modules. On `PS-262.9437.196` it lists these mandatory ids: `com.intellij.modules.xml`,
`com.intellij.modules.ultimate`, `com.intellij.modules.php-capable`,
`com.intellij.modules.json`, `com.intellij.modules.spellchecker`,
`com.intellij.modules.regexp`. The `<module name="...">` entries beside them are platform
content modules ([setup-plugin-model-v2.md](setup-plugin-model-v2.md)); older PHP builds
write the same dependencies as `<depends>` elements instead.

Not every id is yours to declare. A `com.intellij.modules.*` id can be a capability module
the platform choice already satisfies, or a real bundled plugin that needs its own
`bundledPlugin(...)`. Tell them apart in the distribution rather than by name — of those
six, only `com.intellij.modules.json` is a real plugin (it is the JSON plugin's own `<id>`,
`Contents/plugins/json/lib/intellij.json.jar`):

```bash
for j in "$PHPSTORM_HOME"/Contents/plugins/*/lib/*.jar; do
  unzip -p "$j" META-INF/plugin.xml 2>/dev/null | grep -m1 '<id>com.intellij.modules.'
done
```

**The symptom of a missing one is in the sandbox log, not the build output** — `idea.log`
under `.intellijPlatform/sandbox/<name>/<IDE>/log-test/` reads `Plugin 'PHP'
(com.jetbrains.php) has dependency on 'com.intellij.modules.json' which is not installed`,
then `requires plugin with id=com.jetbrains.php to be enabled`. Declare the named id,
rerun, read the log again — one line per missing dependency.

Where the `com.jetbrains.php.*` sources themselves live, and how to read them, is
[php-api-sources.md](php-api-sources.md).

Reference: `$PHPSTORM_HOME/Contents/plugins/php-impl/lib/php.jar!/META-INF/plugin.xml`;
`$PHPSTORM_HOME/Contents/plugins/json/lib/intellij.json.jar!/META-INF/plugin.xml`.

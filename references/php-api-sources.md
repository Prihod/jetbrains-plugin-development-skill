---
title: Locating PhpStorm API sources
tags: php, phpstorm, sources, verification
verify: PHPSTORM_HOME="${PHPSTORM_HOME:?}"; JAR="$PHPSTORM_HOME/Contents/plugins/php/lib/php-openapi-src.jar"; test "$(unzip -l "$JAR" | grep -c 'com/jetbrains/php/')" = "$(sed -n 's/.*holds \([0-9][0-9]*\) files under.*/\1/p' references/php-api-sources.md)" && test "$(unzip -l "$PHPSTORM_HOME/Contents/plugins/php-impl/lib/php-openapi.jar" | grep -c '\.java$')" = 0
---

## Locating PhpStorm API sources

**Verified fact:** the PHP plugin is closed-source and absent from `intellij-community`
— see [source-lookup.md](source-lookup.md#finding-sources). The only authoritative
source for `com.jetbrains.php.*` is the sources jar bundled inside an installed
PhpStorm:

```
$PHPSTORM_HOME/Contents/plugins/php/lib/php-openapi-src.jar
```

Set `PHPSTORM_HOME` the same way this skill sets `$IJ_SRC` / `$IJ_SAMPLES` /
`$PLUGIN_TEMPLATE_HOME` — an explicit environment variable pointing at the installed
PhpStorm (the `.app` bundle on macOS; the install directory on Linux/Windows). Never
hardcode the literal path in prose or in a `verify` command — resolve it from
`$PHPSTORM_HOME`, and fail loudly with `${PHPSTORM_HOME:?}` if it isn't set, so the
file still works on another machine. A sibling jar, `php-impl/lib/php-openapi.jar`, is
compiled classes with no `.java` sources — the wrong one to read.

The archive holds 248 files under `com/jetbrains/php/` (`.java` and `.kt` sources).
Read it:

```bash
PHPSTORM_HOME="${PHPSTORM_HOME:?}"
JAR="$PHPSTORM_HOME/Contents/plugins/php/lib/php-openapi-src.jar"
unzip -l "$JAR"                                  # list every class
unzip -p "$JAR" com/jetbrains/php/PhpIndex.java   # read one source file
```

A class name that this listing doesn't produce is hallucinated, not a gap in the
archive — see [php-psi-index.md](php-psi-index.md) for the entry points this skill has
confirmed that way.

Reference: `$PHPSTORM_HOME/Contents/plugins/php/lib/php-openapi-src.jar`;
`$PHPSTORM_HOME/Contents/plugins/php-impl/lib/php.jar!/META-INF/plugin.xml`.

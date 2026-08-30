---
title: Work with PHP PSI and PhpIndex
tags: php, phpstorm, psi, index
verify: PHPSTORM_HOME="${PHPSTORM_HOME:?}"; JAR="$PHPSTORM_HOME/Contents/plugins/php/lib/php-openapi-src.jar"; total=0; missing=0; for c in $(sed '1,/^---$/d' references/php-psi-index.md | grep -oE '`Php[A-Za-z]+`' | tr -d '`' | sort -u); do total=$((total+1)); unzip -l "$JAR" | grep -q "/$c\.java$" || { echo "MISSING: $c"; missing=$((missing+1)); }; done; test "$total" -gt 0 -a "$missing" -eq 0 || exit 1; X="$PHPSTORM_HOME/Contents/plugins/php-impl/lib/php.jar"; tags=$(unzip -p "$X" META-INF/plugin.xml | tr '\n' ' '); li=$(printf '%s' "$tags" | grep -o '<localInspection[^>]*>'); cc=$(printf '%s' "$tags" | grep -o '<completion.contributor[^>]*>'); n_all=$(printf '%s\n' "$li" | grep -c .); n_php=$(printf '%s\n' "$li" | grep -c 'language="PHP"'); c_all=$(printf '%s\n' "$cc" | grep -c .); c_php=$(printf '%s\n' "$cc" | grep -c 'language="PHP"'); n_json=$(printf '%s\n' "$li" | grep -c 'language="JSON"'); n_re=$(printf '%s\n' "$li" | grep -c 'language="RegExp"'); n_js=$(printf '%s\n' "$li" | grep -c 'language="JavaScript"'); n_rest=$((n_all-n_php)); test "$n_php" -lt "$n_all" || exit 1; test $((n_json+n_re+n_js)) -eq "$n_rest" || exit 1; body=$(sed '1,/^---$/d' references/php-psi-index.md); norm=$(printf '%s' "$body" | tr '\n' ' ' | tr -s ' '); for s in "of the $n_all \`<localInspection>\` entries" "$n_php carry" "the other $n_rest are $n_json \`JSON\`, $n_re \`RegExp\` and $n_js \`JavaScript\`" "$c_php of $c_all \`<completion.contributor>\`" "# $n_php"; do printf '%s' "$norm" | grep -qF "$s" || exit 1; done
---

## Work with PHP PSI and PhpIndex

PHP PSI is a separate parsed tree from Java PSI: PHP files never produce
`PsiJavaFile` or a `com.intellij.psi.PsiClass`. Every PHP-specific element implements
`com.jetbrains.php.lang.psi.elements.PhpPsiElement` (extends `NavigatablePsiElement`),
and every named one also implements `PhpNamedElement`
(`com/jetbrains/php/lang/psi/elements/PhpNamedElement.java`, extends
`PsiNameIdentifierOwner` and `PhpPsiElement`). `PhpFile`
(`com/jetbrains/php/lang/psi/PhpFile.java`) still extends the platform's `PsiFile` —
the layer model in [model-psi.md](model-psi.md) applies unchanged; this file only adds
what's PHP-specific. Concrete elements (`PhpClass`, `Method`, `Function`, `Field`,
`Variable`, `PhpNamespace`, `Constant`, …) live under
`com.jetbrains.php.lang.psi.elements.**`, one branch of the 158 entries confirmed under
`com/jetbrains/php/lang/psi/` (`elements/`, `resolve/`, `stubs/`, `visitors/`).

**`PhpIndex`** (`com/jetbrains/php/PhpIndex.java`) is the entry point for PHP resolve.
`PhpIndex.getInstance(project)` returns the project's instance; `getClassesByFQN(fqn)`,
`getClassByName(name)` and `getClassesByName(name)` answer "which classes match this
name" from the index — the PHP analogue of `JavaPsiFacade.findClass` (see
[model-psi.md](model-psi.md)). The same `DumbService.isDumb(project)` guard applies
before calling into it.

**`PhpClassHierarchyUtils`** (`com/jetbrains/php/PhpClassHierarchyUtils.java`) walks the
PHP class/trait/interface hierarchy: `isSuperClass(PhpClass, PhpClass, boolean)`,
`processSuperClasses(...)`, `processMethods(...)`, `processOverridingMethods(...)` —
PHP's traits and interface multi-inheritance have no equivalent in
`PsiClass.getSupers()`.

**Traversal.** `PhpRecursiveElementVisitor`
(`com/jetbrains/php/lang/psi/visitors/PhpRecursiveElementVisitor.java`) is
`@Deprecated` — its own Javadoc prefers control-flow traversal or PSI utilities for
finding one element, and non-recursive visiting via `PhpElementVisitor`
(`com/jetbrains/php/lang/psi/visitors/PhpElementVisitor.java`) otherwise. Check
`@Deprecated` before using either, per [source-lookup.md](source-lookup.md).

**Control flow.** `com.jetbrains.php.codeInsight.controlFlow.**` (33 entries:
`PhpControlFlow`, `PhpInstructionProcessor`, an `instructions/` subpackage) builds the
instruction graph PHP inspections use for reachability and flow-sensitive checks.

**Inspections and completion.** PHP registers against the platform's own extension
points — `localInspection`, `completion.contributor` — scoped with a `language="PHP"`
attribute. **Verified fact** (from the PHP plugin's own *compiled* descriptor, not the
sources archive the rest of this file cites): in
`$PHPSTORM_HOME/Contents/plugins/php-impl/lib/php.jar!/META-INF/plugin.xml`, of the 391
`<localInspection>` entries 383 carry `language="PHP"` (the other 8 are 4 `JSON`, 2
`RegExp` and 2 `JavaScript` inspections merely grouped under PHP), and 13 of 15
`<completion.contributor>` entries do. Count whole tags: `language` is not always the
first attribute, so the obvious line-prefix grep silently undercounts.

```bash
unzip -p "$PHPSTORM_HOME/Contents/plugins/php-impl/lib/php.jar" META-INF/plugin.xml |
  tr '\n' ' ' | grep -o '<localInspection[^>]*>' | grep -c 'language="PHP"'
# 383   — while grep -c '<localInspection language="PHP"' reports 381
```

`PhpLanguage` (`com/jetbrains/php/lang/PhpLanguage.java`, extends `Language`) is
constructed with that literal ID. See
[editor-inspections-completion.md](editor-inspections-completion.md) for the
platform-side registration mechanics, which this file does not repeat.

Never write a `com.jetbrains.php.*` class name from memory — confirm it in the archive
first, per [php-api-sources.md](php-api-sources.md).

Reference: `com/jetbrains/php/PhpIndex.java`; `com/jetbrains/php/PhpClassHierarchyUtils.java`;
`com/jetbrains/php/lang/psi/elements/PhpNamedElement.java`;
`com/jetbrains/php/lang/psi/visitors/PhpRecursiveElementVisitor.java`;
`com/jetbrains/php/lang/psi/visitors/PhpElementVisitor.java`; `com/jetbrains/php/lang/PhpLanguage.java`;
`$PHPSTORM_HOME/Contents/plugins/php-impl/lib/php.jar!/META-INF/plugin.xml`.

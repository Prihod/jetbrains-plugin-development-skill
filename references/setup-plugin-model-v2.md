---
title: Content modules vs. legacy depends
tags: plugin-xml, plugin-model-v2, content-modules
verify: IJ_SRC="${IJ_SRC:?}"; f=references/setup-plugin-model-v2.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); X="$IJ_SRC/plugins/tasks/tasks-core/resources/META-INF/plugin.xml"; for s in '<content namespace="jetbrains">' '<module name="intellij.tasks.jira"/>' '<module name="intellij.platform.tasks"/>'; do printf '%s\n' "$body" | grep -qF -- "$s" || exit 1; grep -qF -- "$s" "$X" || exit 1; done; grep -q '<dependencies>' "$X"
---

## Content modules vs. legacy `<depends>`

`<depends>` declares a coarse, whole-plugin requirement on another plugin or platform
module — it is still correct for an ordinary plugin, and the template itself uses only
this form. Plugin Model V2 adds `<content>`/`<module>` for a different problem:
splitting **your own** plugin into separately loadable pieces, each with its own
`<dependencies>`.

**Wrong:**

```xml
<!-- plugin.xml — "V2 replaced <depends>, so rewrite it" -->
<dependencies>
    <plugin id="com.intellij.modules.platform"/>
</dependencies>
```

For a plugin with no content modules of its own, this is not what `<dependencies>` is
for and is not how any shipped plugin does it — keep `<depends>` for that case.

**Right** (the real form, from `com.intellij.tasks`'s `plugin.xml` — it ships three
integrations as independently loadable modules):

```xml
<content namespace="jetbrains">
  <module name="intellij.tasks.jira"/>
</content>
<dependencies>
  <plugin id="com.intellij.modules.xml"/>
  <module name="intellij.platform.tasks"/>
</dependencies>
```

- `<content namespace="...">` — in the *main* descriptor, lists the content modules
  this plugin ships; each `<module name="...">` points at a separate descriptor file
  (e.g. `intellij.tasks.jira.xml`) with its own `<extensions>`, loaded only when
  needed.
- `<dependencies>` — a new element, distinct from `<depends>`; inside a content
  module's own descriptor (or the main one) it lists what that module needs, as
  `<plugin id="..."/>` or `<module name="..."/>`.

**When each applies:** an ordinary single-module plugin uses `<depends>` only — this
is the template's form and the common case. Reach for `<content>`/`<dependencies>`
only when splitting your plugin into independently loadable modules; it is not a
drop-in replacement for `<depends>` in a plugin that has no such modules.

Reference: `$IJ_SRC/plugins/tasks/tasks-core/resources/META-INF/plugin.xml`.

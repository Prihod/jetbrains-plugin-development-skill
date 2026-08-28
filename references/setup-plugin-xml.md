---
title: The minimal plugin descriptor
tags: plugin-xml, descriptor
verify: PLUGIN_TEMPLATE_HOME="${PLUGIN_TEMPLATE_HOME:?}"; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' references/setup-plugin-xml.md); for p in 'ForbiddenPluginIdPrefix' 'TemplateWordInPluginName'; do printf '%s\n' "$body" | grep -qF "$p" || exit 1; done && grep -n '<depends>' "$PLUGIN_TEMPLATE_HOME/src/main/resources/META-INF/plugin.xml"
---

## The minimal plugin descriptor

`plugin.xml` is the one file the platform reads before any of your code runs. Every
element below is present in `intellij-platform-plugin-template`'s descriptor; nothing
here is invented.

**Wrong:**

```xml
<!-- plugin.xml -->
<idea-plugin>
    <id>com.acme.devtools</id>
    <name>Acme DevTools</name>
    <vendor>Me</vendor>
    <!-- no <depends> — "Plugin Model V2 replaced it, so it's optional now" -->
</idea-plugin>
```

Omitting `<depends>` does not fail the build; the plugin just never declares which
platform module it targets, which the verifier and the Marketplace both check.

Two more rules the Verifier enforces on `<id>`/`<name>` specifically, neither shown
above since that example is wrong for a different reason: a `com.example`-style
`<id>` prefix trips `ForbiddenPluginIdPrefix`, and a `<name>` containing the word
"plugin" or "template" trips `TemplateWordInPluginName` — confirmed by running
`./gradlew verifyPlugin` on a descriptor using either; the verifier's own output
names the rule. The **Right** block's `<id>`/`<name>` below are the template's own,
unrenamed — copy the structure, not those two values, into a real plugin, or a
`verifyPlugin` run will reject them on both rules.

**Right:**

```xml
<!-- plugin.xml -->
<idea-plugin>
    <id>org.jetbrains.plugins.template</id>
    <name>IntelliJ Platform Plugin Template</name>
    <vendor>JetBrains</vendor>
    <description><![CDATA[ ... ]]></description>

    <depends>com.intellij.modules.platform</depends>

    <resource-bundle>messages.MyBundle</resource-bundle>

    <extensions defaultExtensionNs="com.intellij">
        <toolWindow factoryClass="..." id="MyToolWindow"/>
        <postStartupActivity implementation="..."/>
    </extensions>
</idea-plugin>
```

**Elements:**

- `<id>` — unique, reverse-DNS-style, permanent once published; the Marketplace
  identity.
- `<name>` — display name shown in Settings > Plugins and the Marketplace.
- `<vendor>` — author or organization, shown next to the plugin.
- `<description>` — CDATA HTML shown in the Marketplace and the Plugins list.
- `<depends>` — declares what this plugin requires: a platform module
  (`com.intellij.modules.platform` is the baseline every plugin needs) or another
  plugin's id. See `setup-dependency-origin.md` for choosing what goes here.
- `<resource-bundle>` — base name of the `.properties` file backing i18n message
  lookups (`MyBundle.message(...)`).
- `<extensions defaultExtensionNs="com.intellij">` — where extension point
  registrations live; empty is valid, absent is not if you register anything.

Compatibility range (`since-build`/`until-build`) is not hand-written here — current
tooling fills it in; see `antipatterns-build.md` for what else is deliberately absent.

Reference: `intellij-platform-plugin-template`'s
`src/main/resources/META-INF/plugin.xml`.

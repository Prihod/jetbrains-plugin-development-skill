---
title: Bind a settings page to state with BoundConfigurable, not manual sync
tags: ui, settings, configurable, persistent-state-component
verify: IJ_SRC="${IJ_SRC:?}"; f=references/ui-settings.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'PersistentStateComponent<T>' || exit 1; printf '%s\n' "$body" | grep -qF 'BoundConfigurable(' || exit 1; grep -n "public interface PersistentStateComponent<T> {" "$IJ_SRC/platform/projectModel-api/src/com/intellij/openapi/components/PersistentStateComponent.java" && grep -n "abstract class BoundConfigurable(" "$IJ_SRC/platform/platform-api/src/com/intellij/openapi/options/BoundConfigurable.kt"
---

## Bind a settings page to state with BoundConfigurable, not manual sync

State storage and the settings UI are two separate contracts. `PersistentStateComponent<T>`
(`platform/projectModel-api/.../components/PersistentStateComponent.java`) persists a
plain state object via `getState()`/`loadState(T)`, driven by `@State`
(`.../components/State.java`) and `@Storage` (`.../components/Storage.java`) on
the component class. `Configurable`
(`platform/ide-core/src/com/intellij/openapi/options/Configurable.java`, extends
`UnnamedConfigurable`) is the page shown in Settings; nothing forces its `isModified()` /
`apply()` / `reset()` to actually agree with the state object.

**Wrong (raw `Configurable` — UI and state drift independently):**

```java
public boolean isModified() {
    State state = AppSettings.getInstance().getState();
    return !component.getUserNameText().equals(state.userId); // easy to miss a field here
}
public void apply() {
    AppSettings.getInstance().getState().userId = component.getUserNameText(); // and here
}
```

**Right (`BoundConfigurable` + UI DSL bindings — one place declares the link):**

```kotlin
class MySettingsConfigurable : BoundConfigurable("My Plugin") {
    override fun createPanel() = panel {
        row("User ID:") {
            textField().bindText(AppSettings.getInstance().state::userId)
        }
    }
}
```

`BoundConfigurable` (`platform/platform-api/.../options/BoundConfigurable.kt`,
abstract class) wires `panel { }`'s bindings into `isModified()`/`apply()`/`reset()`
for you — see [ui-dsl-dialogs.md](ui-dsl-dialogs.md) for `panel`/`row`/`cell`/`bindText`
themselves. `settings`'s `AppSettings`/`AppSettingsConfigurable`
(`$IJ_SAMPLES/settings/src/main/java/org/intellij/sdk/settings/`, `settings` in
[source-lookup-samples.md](source-lookup-samples.md)) shows the raw, unbound form
above verbatim; `oauth2`'s `AuthConfigurable` shows the `BoundConfigurable` + `panel { }`
form. A page that needs `focusOn(@NotNull @Nls String label)`
(`Configurable.java`) to jump to one field is a signal it grew past one screen —
split it rather than growing `createPanel()` further.

## Registration and migration

`applicationConfigurable`/`projectConfigurable` are both declared with `name=` in
`platform/platform-resources/src/META-INF/PlatformExtensionPoints.xml` — search both
`name="..."` and `qualifiedName="...<EP>"` forms before
concluding either is invented (see
[source-lookup.md](source-lookup.md#search-recipes)). Renaming a field or bumping
`@Storage`'s file changes what `loadState` receives on old data; deserialize
defensively (nullable/defaulted fields) rather than assuming every saved field is
still present.

Reference: `platform/projectModel-api/src/com/intellij/openapi/components/PersistentStateComponent.java`,
`State.java`, `Storage.java`;
`platform/ide-core/src/com/intellij/openapi/options/Configurable.java`,
`UnnamedConfigurable.java`;
`platform/platform-api/src/com/intellij/openapi/options/BoundConfigurable.kt`;
`platform/platform-resources/src/META-INF/PlatformExtensionPoints.xml`;
`$IJ_SAMPLES/settings/src/main/java/org/intellij/sdk/settings/`;
`$IJ_SAMPLES/oauth2/src/main/kotlin/org.intellij.sdk.oauth2/AuthConfigurable.kt`.

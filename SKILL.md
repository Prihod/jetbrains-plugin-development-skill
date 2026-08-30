---
name: jetbrains-plugin-development
description: >-
  Develop, debug, test and maintain plugins for IntelliJ Platform IDEs — IntelliJ IDEA,
  PhpStorm, WebStorm, PyCharm, GoLand. Covers plugin.xml, extension points, services,
  actions, PSI, VFS, threading and lifecycle, the IntelliJ Platform Gradle Plugin 2.x
  (org.jetbrains.intellij.platform), plugin verification and publishing. Use when creating
  or modifying an IntelliJ Platform plugin project, or when asked about IntelliJ Platform APIs.
targets: [claude, codex]
metadata:
  short-description: "Build IntelliJ Platform plugins against verified APIs"
  author: Prihod
  source: https://github.com/Prihod/jetbrains-plugin-development-skill
---

# JetBrains Plugin Development

## Baseline

Skill version 1.0.0 — baseline validated 2026-08-26 against the official `intellij-platform-plugin-template`.
These are the current defaults, not the ones most tutorials show.

| What | Value |
|---|---|
| IntelliJ Platform Gradle Plugin | 2.16.0, applied as `org.jetbrains.intellij.platform.settings` in `settings.gradle.kts` |
| Gradle | 9.5.0 |
| Kotlin JVM plugin | 2.1.20 |
| `org.jetbrains.changelog` | 2.5.0 |
| Platform dependency | `intellijIdea("2025.2.6.2")` inside `dependencies { intellijPlatform { } }` |
| `kotlin.stdlib.default.dependency` | `false` |

The project you are working in wins over this table. If versions differ, follow the project.

The sources you read and the build you compile against are two artefacts with two build numbers — on the validated setup `$IJ_SRC` is `intellij-community` branch 262 while `intellijIdea("2025.2.6.2")` resolves to `IU-252.28539.54`, and yours may pair differently. Check every symbol at the target — [references/source-lookup-target-build.md](references/source-lookup-target-build.md).

## Step 0 — detect the target

Do this before changing anything. An API existing in IntelliJ IDEA does not mean it exists in the target IDE. Read it from `settings.gradle.kts`, `build.gradle.kts`, `gradle.properties`, `gradle/wrapper/gradle-wrapper.properties` and `plugin.xml` — never infer it from code style.

IDE and version · platform version · build number (since/until) · JDK · Kotlin · Gradle · IntelliJ Platform Gradle Plugin · minimum supported IDE · bundled and external plugin dependencies.

## Critical rules

1. Verify unfamiliar APIs against sources first — [source-lookup](references/source-lookup.md) — then confirm at the target build that the symbol exists and is not deprecated; sources run ahead of it.
2. Every action declares `getActionUpdateThread()`; the default is `EDT` and the compiler won't warn you — [traps](references/antipatterns-edt-and-actions.md).
3. `update()` stays cheap: no PSI resolve, no I/O, no indexing on the EDT.
4. Every resource has one explicit lifecycle owner — a `Disposable` or a coroutine scope.
5. Read the code model under a read action; write under a write action on the EDT.
6. Know a dependency's origin — platform, bundled plugin, or external plugin — and declare it in both the build and `plugin.xml`.
7. Files belonging to the project go through VFS, not `java.io.File`.
8. Look for an existing extension point before declaring your own.
9. Deprecated API is replaced, not suppressed; if it must stay, record why, affected versions, the alternative, and the removal plan.
10. Domain logic lives in a service, never in `AnAction`, `ToolWindowFactory` or a UI class.
11. Never claim compatibility you have not verified with the plugin verifier.
12. Publishing is external and hard to reverse — only on explicit user confirmation.

## Verifying an unfamiliar API

Locate sources → find the declaration → check `@Deprecated` / `@ApiStatus.*` → check the threading contract (`@RequiresEdt`, `@RequiresReadLock`, `@RequiresBackgroundThread`) → find a real usage in platform tests or JetBrains samples → determine the dependency's origin → confirm availability in the target IDE and version → implement → compile and test.

Do not skip these because the API "looks obvious" — obvious APIs are the ones that get renamed and removed. With no sources available, say so and lower your confidence; do not fill the gap with a guess.

## Navigation

| Task | File |
|---|---|
| Create a new plugin project | [references/workflow-create-plugin.md](references/workflow-create-plugin.md) |
| Add an action, service, tool window, settings page or inspection | [references/workflow-add-feature.md](references/workflow-add-feature.md) |
| Verify an API, locate platform sources | [references/source-lookup.md](references/source-lookup.md) |
| Find a JetBrains sample for a task | [references/source-lookup-samples.md](references/source-lookup-samples.md) |
| Check an API against the target build, not the sources | [references/source-lookup-target-build.md](references/source-lookup-target-build.md) |
| EDT and action-update-cycle traps | [references/antipatterns-edt-and-actions.md](references/antipatterns-edt-and-actions.md) |
| PSI and VFS traps | [references/antipatterns-psi-vfs.md](references/antipatterns-psi-vfs.md) |
| Lifecycle and dumb-mode traps | [references/antipatterns-lifecycle-dumb-mode.md](references/antipatterns-lifecycle-dumb-mode.md) |
| Deprecated and legacy platform API traps | [references/antipatterns-api-deprecated.md](references/antipatterns-api-deprecated.md) |
| Extension point traps | [references/antipatterns-extension-points.md](references/antipatterns-extension-points.md) |
| Build configuration traps | [references/antipatterns-build.md](references/antipatterns-build.md) |
| Undeclared bundled-plugin dependency traps | [references/antipatterns-dependencies.md](references/antipatterns-dependencies.md) |
| Set up or fix the plugin build | [references/setup-build.md](references/setup-build.md) |
| Write the plugin descriptor | [references/setup-plugin-xml.md](references/setup-plugin-xml.md) |
| Use content modules (Plugin Model V2) | [references/setup-plugin-model-v2.md](references/setup-plugin-model-v2.md) |
| Decide where a dependency comes from | [references/setup-dependency-origin.md](references/setup-dependency-origin.md) |
| Find or declare an extension point | [references/extend-extension-points.md](references/extend-extension-points.md) |
| Add a service; keep logic out of the UI | [references/extend-services.md](references/extend-services.md) |
| Add an action | [references/extend-actions.md](references/extend-actions.md) |
| Add a listener; keep the plugin dynamic | [references/extend-listeners-dynamic.md](references/extend-listeners-dynamic.md) |
| Decide which thread to run on | [references/threading-model.md](references/threading-model.md) |
| Read or write the code model safely | [references/threading-read-write.md](references/threading-read-write.md) |
| Use coroutines without leaking scopes | [references/threading-coroutines.md](references/threading-coroutines.md) |
| Pick a constructor the container can call | [references/threading-service-constructor-shapes.md](references/threading-service-constructor-shapes.md) |
| Own a lifecycle; subscribe and unsubscribe | [references/lifecycle-disposable-messagebus.md](references/lifecycle-disposable-messagebus.md) |
| Work with PSI, indexes and dumb mode | [references/model-psi.md](references/model-psi.md) |
| Work with virtual files | [references/model-vfs.md](references/model-vfs.md) |
| Work with the project and modules | [references/model-project.md](references/model-project.md) |
| Work with the editor and its document | [references/editor-document-listeners.md](references/editor-document-listeners.md) |
| Add an inspection, intention, completion or inlay hint | [references/editor-inspections-completion.md](references/editor-inspections-completion.md) |
| Build a dialog, popup or notification | [references/ui-dsl-dialogs.md](references/ui-dsl-dialogs.md) |
| Add a tool window | [references/ui-tool-windows.md](references/ui-tool-windows.md) |
| Add a settings page | [references/ui-settings.md](references/ui-settings.md) |
| Add icons and localized strings | [references/ui-icons-i18n.md](references/ui-icons-i18n.md) |
| Pick a test level and wire fixtures | [references/testing-levels-fixtures.md](references/testing-levels-fixtures.md) |
| Test PSI and the editor | [references/testing-psi-editor.md](references/testing-psi-editor.md) |
| Test an action's `update()` and data context | [references/testing-actions.md](references/testing-actions.md) |
| Target multiple IDE versions and verify | [references/compat-range-and-verifier.md](references/compat-range-and-verifier.md) |
| Bind the verifier's IDE; know what it downloads | [references/compat-verifier-ides-and-cost.md](references/compat-verifier-ides-and-cost.md) |
| Replace a deprecated API; read logs | [references/compat-deprecated-policy.md](references/compat-deprecated-policy.md) |
| Write tests; reproduce and diagnose a bug | [references/workflow-test-and-debug.md](references/workflow-test-and-debug.md) |
| Raise the platform version or migrate from 1.x | [references/workflow-upgrade-platform.md](references/workflow-upgrade-platform.md) |
| Verify, sign and publish | [references/workflow-release-plugin.md](references/workflow-release-plugin.md) |
| Depend on the PHP plugin and its own dependency chain | [references/php-dependency-sources.md](references/php-dependency-sources.md) |
| Find and read PhpStorm API sources | [references/php-api-sources.md](references/php-api-sources.md) |
| Work with PHP PSI and PhpIndex | [references/php-psi-index.md](references/php-psi-index.md) |

## Validating your changes

Never assume Gradle task names. List what the project actually offers first:

    ./gradlew tasks

Then run the project's build, tests and plugin verification.

---
title: JetBrains sample map — task to sample directory
tags: sources, samples, examples
verify: IJ_SAMPLES="${IJ_SAMPLES:?}"; f=references/source-lookup-samples.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); total=0; missing=0; for d in $(printf '%s\n' "$body" | grep -oE '`[a-z0-9_]+`' | tr -d '`' | sort -u); do total=$((total+1)); [ -d "$IJ_SAMPLES/$d" ] || { echo "MISSING: $d"; missing=$((missing+1)); }; done; test "$total" -ge 21 -a "$missing" -eq 0
---

## Sample map

Each row names a directory in an `intellij-sdk-code-samples` checkout — see
[Finding sources](source-lookup.md#finding-sources) for how to locate one. The skill
contains no samples of its own; treat this table as an index into that checkout, not
as a copy of its content.

| Task | Sample directory |
|---|---|
| Register an action, group, or shortcut | `action_basics` |
| Write a Qodana-compatible code inspection | `code_inspection_qodana` |
| Compare string/object references safely (inspection pattern) | `comparing_string_references_inspection` |
| Write a code intention (local quick-fix-style transform) | `conditional_operator_intention` |
| Work with the editor: carets, selections, documents | `editor_basics` |
| Add a facet type to a module | `facet_basics` |
| Support a framework (framework detection/integration) | `framework_basics` |
| Define live templates | `live_templates` |
| Limit or track the number of open projects (plugin service) | `max_opened_projects` |
| Add a custom module type (project wizard) | `module` |
| Add an OAuth2 login flow (PKCE, PasswordSafe) | `oauth2` |
| Work with the project model: SDKs, libraries, project structure | `project_model` |
| Add a custom pane to the Project view | `project_view_pane` |
| Add a step or provider to the New Project wizard | `project_wizard` |
| Navigate or manipulate the PSI tree | `psi_demo` |
| Add a custom run configuration type | `run_configuration` |
| Add a Settings/Preferences page | `settings` |
| Build a custom language plugin (lexer, parser, PSI) | `simple_language_plugin` |
| Ship a custom UI theme | `theme_basics` |
| Add a tool window | `tool_window` |
| Customize the Project view's tree structure | `tree_structure_provider` |

## Locating the checkout

See [source-lookup.md#finding-sources](source-lookup.md#finding-sources) for the
search order used to locate an `intellij-sdk-code-samples` checkout. If none is
found, fall back to the official documentation and say so out loud.

Reference: `intellij-sdk-code-samples` checkout root.

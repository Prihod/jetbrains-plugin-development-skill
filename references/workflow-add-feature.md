---
title: Add an action, service, tool window, settings page or inspection
tags: workflow, actions, services, extension-points, ui, testing
verify: IJ_SAMPLES="${IJ_SAMPLES:?}"; for pair in extend-actions.md:action_basics extend-services.md:max_opened_projects ui-tool-windows.md:tool_window ui-settings.md:settings editor-inspections-completion.md:comparing_string_references_inspection editor-inspections-completion.md:conditional_operator_intention; do r=${pair%%:*}; s=${pair##*:}; re=$(printf %s "$r" | sed 's/\./\\./g'); se=$(printf %s "$s" | sed 's/\./\\./g'); grep -Eq "\|[[:space:]]*\`$re\`[[:space:]]*\|[[:space:]]*\`$se\`[[:space:]]*\|" references/workflow-add-feature.md && [ -f "references/$r" ] && [ -d "$IJ_SAMPLES/$s" ] || exit 1; done
---

## Add an action, service, tool window, settings page or inspection

One procedure serves every feature addition — an action, a service, a tool window, a settings page, an inspection, an intention. Only the extension mechanism and its reference differ; there is no separate `add-action`/`add-service`/`add-extension` workflow.

### Preconditions

- A plugin project that already builds — see [workflow-create-plugin.md](workflow-create-plugin.md).
- The feature is scoped to one extension mechanism from the table below; a feature spanning several (e.g. an action that opens a settings page) runs this procedure once per mechanism.

### Steps

1. Run Step 0 (detect) if you have not already
2. Pick the extension mechanism
3. Look for an existing extension point before declaring your own
4. Verify the API you are about to use
5. Put the logic in a service, not in the UI class
6. Register it in plugin.xml
7. Answer the three concurrency questions: which thread, who owns it, who cancels it
8. Write the test
9. Build, test, verify

Step 1 is `SKILL.md`'s Step 0 — detect the target. Step 2 picks a row below. Step 3 is [extend-extension-points.md](extend-extension-points.md) — reuse before you reinvent (AP-13). Step 4 is [source-lookup.md](source-lookup.md)'s Finding sources order. Step 5 follows [extend-services.md#layering](extend-services.md#layering) — `UI → Service → Domain`; give any resource it owns a `Disposable` (AP-06). Step 6 declares the extension and its dependency origin together — an API from a bundled plugin needs both a `<depends>` and the matching build dependency, or it loads in the IDE you built against and fails everywhere else (AP-12). Step 7 is [threading-model.md](threading-model.md) — which thread runs it, who owns the `Disposable`/scope, what calls `checkCanceled()`. Step 8 is [testing-levels-fixtures.md](testing-levels-fixtures.md) — the lowest level that proves the behavior. Step 9 is [workflow-create-plugin.md#validation](workflow-create-plugin.md#validation) — list task names first, then build, test, and re-verify if `plugin.xml`'s compatibility footprint changed.

| Feature | Reference | Sample |
|---|---|---|
| Action | `extend-actions.md` | `action_basics` |
| Service | `extend-services.md` | `max_opened_projects` |
| Tool window | `ui-tool-windows.md` | `tool_window` |
| Settings page | `ui-settings.md` | `settings` |
| Inspection | `editor-inspections-completion.md` | `comparing_string_references_inspection` |
| Intention | `editor-inspections-completion.md` | `conditional_operator_intention` |

### Validation

Same discipline as [workflow-create-plugin.md#validation](workflow-create-plugin.md#validation): list Gradle task names before running any of them. Run the test written in Step 8 first, then the full suite, then plugin verification if the extension point or `<depends>` changed the compatibility footprint.

### Common mistakes

AP-02 (`getActionUpdateThread()` silently defaults to EDT) and AP-03 (PSI `resolve()` inside `update()` while stuck there) hit an action the moment Step 7 is skipped. AP-06 is a service's coroutine scope or message-bus subscription with no `Disposable` owner from Step 5. AP-12 is Step 6's dependency declared in `plugin.xml` but not in the build, or the reverse. AP-13 is Step 3 skipped — a private registry instead of an existing extension point.

### References

[extend-actions.md](extend-actions.md); [extend-services.md](extend-services.md); [ui-tool-windows.md](ui-tool-windows.md); [ui-settings.md](ui-settings.md); [editor-inspections-completion.md](editor-inspections-completion.md); [extend-extension-points.md](extend-extension-points.md); [threading-model.md](threading-model.md); [testing-levels-fixtures.md](testing-levels-fixtures.md); [source-lookup.md](source-lookup.md); [source-lookup-samples.md](source-lookup-samples.md); [workflow-create-plugin.md](workflow-create-plugin.md).

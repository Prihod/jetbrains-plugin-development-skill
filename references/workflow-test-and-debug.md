---
title: Reproduce, diagnose and fix a bug; add a regression test
tags: workflow, debugging, logging, testing
verify: body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' references/workflow-test-and-debug.md); printf '%s\n' "$body" | grep -qFx -- 'Reproduce -> logs -> stack trace -> breakpoint -> state -> threading' && printf '%s\n' "$body" | grep -qFx -- '-> lifecycle -> extension registration -> fix -> regression test' && for ap in AP-03 AP-04 AP-06 AP-14; do printf '%s\n' "$body" | grep -qw -- "$ap" || exit 1; done && grep -q '^## AP-03:' references/antipatterns-edt-and-actions.md && grep -q '^## AP-04:' references/antipatterns-edt-and-actions.md && grep -q '^## AP-06:' references/antipatterns-lifecycle-dumb-mode.md && grep -q '^## AP-14:' references/antipatterns-lifecycle-dumb-mode.md && printf '%s\n' "$body" | grep -qF -- 'Step 10 is a gate, not a bullet.'
---

## Reproduce, diagnose and fix a bug; add a regression test

One cycle handles a crash, a wrong result and a silently inactive extension — it
narrows through the same stages regardless of which one you started with.

### Preconditions

- A plugin project that already builds — see [workflow-create-plugin.md](workflow-create-plugin.md).
- A concrete reproduction: which action, which project state, expected versus actual. A report without a reproduction is a hypothesis, not a bug yet.

### Steps

```
Reproduce -> logs -> stack trace -> breakpoint -> state -> threading
-> lifecycle -> extension registration -> fix -> regression test
```

1. **Reproduce** — run the exact sequence under `Run Plugin` (`$PLUGIN_TEMPLATE_HOME/.run/Run Plugin.run.xml`); a bug that will not reproduce on demand cannot later be confirmed fixed.
2. **Logs** — read `idea.log` from the sandbox that run configuration used; see [compat-deprecated-policy.md](compat-deprecated-policy.md)'s Logging and diagnosis section for where it lands and why the path varies by run configuration.
3. **Stack trace** — read top to bottom for the first frame inside your own plugin's package; platform frames above it are context, not the cause.
4. **Breakpoint** — set it at that frame; step to see live values, not the ones assumed.
5. **State** — check dumb mode, write-safe context and thread identity there; a value being null is often a consequence of one of these, not the root cause.
6. **Threading** — confirm the actual thread against [threading-model.md](threading-model.md); a value reached from the wrong thread is AP-03/AP-04 territory.
7. **Lifecycle** — confirm the resource in play (listener, coroutine scope, subscription) has a live `Disposable` owner — [lifecycle-disposable-messagebus.md](lifecycle-disposable-messagebus.md); a callback that fires unexpectedly or never fires is often AP-06's leaked or wrongly-scoped owner.
8. **Extension registration** — if nothing above explains it, re-check `plugin.xml`: exact extension point name, `implementation`/`implementationClass`, `<depends>` for the declaring plugin — see [setup-plugin-xml.md](setup-plugin-xml.md).
9. **Fix** — the smallest change addressing the stage found above, not the symptom.
10. **Regression test** — at the lowest level that reproduces the bug, per [testing-levels-fixtures.md](testing-levels-fixtures.md); commit the `Document` before asserting on PSI ([testing-psi-editor.md](testing-psi-editor.md)).

**An extension that does not fire is almost always a registration or dumb-mode problem, not a logic bug.** Check the exact extension point name and attributes in `plugin.xml` before debugging the class's own code, and whether the extension needs `DumbAware` — AP-14.

### Validation

Never assume Gradle task names. List what this project actually offers first:

```
./gradlew tasks
```

Run the Step 10 regression test alone first, then the full test task, using the names `./gradlew tasks` printed for this project.

**Step 10 is a gate, not a bullet.** Do not report a bug fixed until that regression test exists, fails on the pre-fix code and passes on the fixed code — run it against both. A fix with no failing-before test is an assertion, not evidence.

### Common mistakes

AP-03 and AP-04 look like ordinary logic bugs until Step 6 asks which thread you are actually on. AP-06 looks like a leak or a callback that "sometimes doesn't fire" until Step 7 checks the `Disposable` owner. AP-14 is Step 8's dumb-mode case — an extension that works once indexed and throws while reindexing.

### References

[testing-levels-fixtures.md](testing-levels-fixtures.md); [testing-psi-editor.md](testing-psi-editor.md); [threading-model.md](threading-model.md); [lifecycle-disposable-messagebus.md](lifecycle-disposable-messagebus.md); [setup-plugin-xml.md](setup-plugin-xml.md); [antipatterns-edt-and-actions.md](antipatterns-edt-and-actions.md); [antipatterns-lifecycle-dumb-mode.md](antipatterns-lifecycle-dumb-mode.md); [compat-deprecated-policy.md](compat-deprecated-policy.md); [workflow-create-plugin.md](workflow-create-plugin.md).

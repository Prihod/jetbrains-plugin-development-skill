# Changelog

All notable changes to this skill are recorded here. Versions follow `MAJOR.MINOR.PATCH`
independently of the IntelliJ Platform's own versions: MAJOR for structural or incompatible
change, MINOR for new topics, workflows or IDEs, PATCH for corrections to text and links.

## 1.0.0 — 2026-08-28

First release.

### Contents

- `SKILL.md`, 119 lines: the pinned build baseline, target detection, 12 critical rules, the
  API-verification procedure, and a navigation table with one row per reference.
- 45 reference files (a 46th, `_template.md`, is the authoring template). By family: 3 `source-*`,
  7 `antipatterns-*`, 4 `setup-*`, 4 `extend-*`, 3 `threading-*`, 1 `lifecycle-*`, 3 `model-*`,
  2 `editor-*`, 4 `ui-*`, 3 `testing-*`, 3 `compat-*`, 5 `workflow-*`, 3 `php-*`.
- Every one of the 45 carries a `verify:` command in its frontmatter. 43 re-derive the file's
  central claim from a first-hand source outside the skill and abort when the variable naming it
  is unset; the remaining two — `workflow-test-and-debug.md` and `workflow-upgrade-platform.md` —
  check only the skill's own files. Each command also anchors to a literal its own body asserts,
  so the check turns red on two independent kinds of rot: the platform moving, and this file no
  longer saying what it says today. The source repository's validator runs all 45 and re-runs each
  against a gutted copy of its own file, requiring red — `45 run, 0 skipped`, with no exception
  list a reference could be parked in.
- `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`: the repository is its own
  Claude Code marketplace, so the skill installs with `/plugin install` as well as by pointing an
  agent's skills directory at the checkout. The skill sits at the repository root, which Claude
  Code detects without a `skills` path in either manifest.
- `agents/openai.yaml`, the Codex interface adapter; `README.md`; Apache License 2.0.

### Baseline

Captured 2026-08-26 from the official `intellij-platform-plugin-template` and re-derivable from a
local checkout of it: IntelliJ Platform Gradle Plugin 2.16.0, Gradle 9.5.0, Kotlin JVM 2.1.20,
`org.jetbrains.changelog` 2.5.0, platform dependency `intellijIdea("2025.2.6.2")`,
`kotlin.stdlib.default.dependency=false`.

### Acceptance

Two exercises, both on 2026-08-27; the full record lives in the project's source repository, not
in the published skill.

- **Live plugin.** An agent given only this skill built a plugin from scratch: build and test
  green, targeting `IU-252.28539.54`. Verification was not green first time — `verifyPlugin`
  **failed**, rejecting the descriptor under `ForbiddenPluginIdPrefix` and
  `TemplateWordInPluginName`; the verdict `Compatible` against an installed IntelliJ IDEA
  `IU-262.9437.185` came only after the descriptor was fixed, and that fix, together with the
  local verifier setup, was made by agents with the full task context rather than by the
  skill-only agent. It exposed four workflow defects, all fixed before release: the undeclared
  ~6.5 GB first Gradle sync; `verifyPlugin` downloading a second IDE distribution into its own
  cache; no documented way to pin the verifier's IDEs offline, the working DSL form having to be
  recovered with `javap`; and a `plugin.xml` example that failed the verifier rules the same file
  teaches.
- **Seven acceptance scenarios**, each run by a fresh agent and judged by a second agent that
  re-checked the artefacts itself. Final result: all seven PASS — after three fix passes.
  Scenarios 6 (compatibility) and 7 (bug fixing) failed their first run and passed only after the
  skill was changed; scenario 2's first run was void through a fixture defect and was rerun on a
  stripped copy. The most serious finding, from scenario 3: `threading-read-write.md` taught
  `ReadAction.computeBlocking`, which does not exist on the platform build this skill pins, and
  called `ReadAction.compute` deprecated when it is not — the skill committing the exact error it
  exists to prevent. Fixed, and the fix confirmed by disassembling the resolved distribution.
  Two further candidate defects did not survive verification and were deliberately not written up.

**Validated on Claude only.** The `agents/openai.yaml` adapter ships and `targets` declares
`codex`, but the skill has never been exercised on Codex. Junie and JetBrains AI Assistant read
these files as ordinary Markdown; no adapter for them is shipped or tested.

### Deviations from the specification the skill was built from

- Antipatterns are split by topic into **seven** files, not the two the specification prescribed
  with a fixed id assignment. Accuracy of file naming beat conformance to the older layout.
- Reference count: the planning artefacts say 36; the shipped, measured count is **45**.
- One navigation row is worded differently from the plan text, so that inlay hints — which
  `editor-inspections-completion.md` covers — are findable in the table.
- The PhpStorm references locate the IDE through `PHPSTORM_HOME` rather than an absolute path, so
  that no machine-specific path ships.

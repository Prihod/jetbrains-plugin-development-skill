# JetBrains Plugin Development

An Agent Skill for building, debugging, testing and maintaining plugins for IntelliJ Platform
IDEs — IntelliJ IDEA, PhpStorm, WebStorm, PyCharm, GoLand.

**Version 1.0.0.** Baseline captured and validated 2026-08-26; acceptance exercises run
2026-08-27, on Claude only.

## What it is for

Coding agents state IntelliJ Platform APIs from memory. The platform is large, long-lived and
renamed often, so a plausible answer is frequently a wrong one — and the compiler only catches
some of it. The premise of this skill is that a platform claim must be traced to a first-hand
source before it becomes code: platform sources, the jars of an installed IDE, the JetBrains
samples, the official plugin template — and then confirmed **at the build the project actually
targets**, which is usually behind the sources you are reading.

Covered: `plugin.xml` and Plugin Model V2, the IntelliJ Platform Gradle Plugin 2.x, extension
points, services, actions, listeners, disposal and lifecycle, threading and the read/write model,
PSI, VFS and the project model, editor features, UI, tests, compatibility ranges and the plugin
verifier, signing and publishing, and PhpStorm/PHP specifics.

That premise is not decoration. This skill's own acceptance exercise caught it teaching
`ReadAction.computeBlocking` as current API — a method that does not exist on the platform build
the skill itself tells you to target — and calling `ReadAction.compute` deprecated when it is not.
The agent that copied that example got `Unresolved reference`. It was found, fixed and re-verified
by disassembling the resolved distribution. See *What was validated* below.

## Install

The repository is both a Claude Code plugin marketplace and the skill itself. Install it either way
— **not both**, or the skill loads twice under the same name.

As a plugin, from inside Claude Code:

    /plugin marketplace add Prihod/jetbrains-plugin-development-skill
    /plugin install jetbrains-plugin-development@prihod

Or point an agent's skills directory straight at the checkout — the repository root **is** the
skill, with `SKILL.md` and `references/` at the top level:

    # personal, for every project
    mkdir -p ~/.claude/skills
    ln -sfn "$PWD" ~/.claude/skills/jetbrains-plugin-development

    # or vendored into one project
    mkdir -p .claude/skills/jetbrains-plugin-development
    git archive HEAD | tar -x -C .claude/skills/jetbrains-plugin-development

The `mkdir -p` is not decoration: on a machine where the skills directory does not exist yet, both
commands fail with `No such file or directory`. The vendored form uses `git archive` rather than
`cp -R` on purpose — `cp -R .` would copy this repository's `.git` directory into your project.
The symlink form is the one this project used for its own acceptance runs. For any other agent,
point it at the same directory. Of the 53 skill files 49 are Markdown; the other four are
`LICENSE`, `agents/openai.yaml` — a three-field interface manifest for Codex — and the two
manifests under `.claude-plugin/` that make this repository installable as a plugin. The vendored
copy also carries the repository's `.gitignore`, since `git archive` exports whatever is
tracked — 54 files land on disk in all. Nothing in the skill is executed on load.

## Optional configuration

Nothing is required. Setting these makes verification cheaper and sharper. Each is read only
where it is used, and the commands that use one fail loudly when it is unset rather than
expanding to an empty path.

| Variable | Points at | What it gives |
|---|---|---|
| `IJ_SRC` | an `intellij-community` checkout | platform implementation, platform tests, threading annotations, extension-point declarations |
| `IJ_SAMPLES` | an `intellij-sdk-code-samples` checkout | 21 compilable JetBrains samples, used as worked examples instead of invented ones |
| `PLUGIN_TEMPLATE_HOME` | an `intellij-platform-plugin-template` checkout | the reference build configuration the Baseline table in `SKILL.md` is derived from and re-checked against |
| `PHPSTORM_HOME` | an installed PhpStorm | the only authoritative source for the closed-source PHP API (`php-openapi-src.jar` inside the IDE) |

`references/source-lookup.md` also names `INTELLIJ_COMMUNITY_HOME` as a variable a project may
already set for the same checkout, but every command in the skill reads `IJ_SRC`.

### It works without any local checkout

With none of these set, the skill still works: it falls back to the official JetBrains
documentation and is required to **say out loud that its confidence is lower** instead of filling
the gap with a guess (`SKILL.md`, *Verifying an unfamiliar API*; `references/source-lookup.md`
step 5). The fallback is a documented degradation, not a silent one — and the verification
commands degrade the same way. One of them used to report 21 missing sample directories and still
exit zero; acceptance found that, and the guards were hardened: 43 of the 45 `verify:` commands now
abort on an unset variable instead of expanding it to an empty path, and the two that do not need
one check only the skill's own files.

## Layout

    SKILL.md             119 lines: baseline table, Step 0, 12 critical rules, navigation
    references/          46 files: 45 references + _template.md
    agents/openai.yaml   Codex interface adapter
    .claude-plugin/      plugin.json and marketplace.json: install as a Claude Code plugin
    README.md
    CHANGELOG.md
    LICENSE

`SKILL.md` is always in context; a reference is read only when the navigation table sends the
agent to it. Every one of the 45 references carries a `verify:` command in its frontmatter — a
command that re-derives the file's own central claim from a real source, so the file can be
checked instead of trusted. (`_template.md` is the authoring template and is excluded from that
set.)

| Prefix | Files | Topic |
|---|---|---|
| `source-*` | 3 | locating sources, JetBrains samples, checking a symbol at the target build |
| `antipatterns-*` | 7 | EDT and actions, PSI/VFS, lifecycle and dumb mode, deprecated API, extension points, build, dependencies |
| `setup-*` | 4 | build, descriptor, Plugin Model V2, dependency origin |
| `extend-*` | 4 | extension points, services, actions, listeners |
| `threading-*` | 3 | threading model, read/write, coroutines |
| `lifecycle-*` | 1 | `Disposable` and the message bus |
| `model-*` | 3 | PSI, VFS, project |
| `editor-*` | 2 | documents and listeners, inspections/intentions/completion/inlay hints |
| `ui-*` | 4 | dialogs, tool windows, settings, icons and i18n |
| `testing-*` | 3 | test levels and fixtures, PSI and editor, actions |
| `compat-*` | 3 | version range and verifier, binding verifier IDEs and their cost, deprecation policy |
| `workflow-*` | 5 | create a plugin, add a feature, test and debug, upgrade the platform, release |
| `php-*` | 3 | PHP plugin dependency, PhpStorm API sources, PHP PSI and `PhpIndex` |

## What was validated, on what, and when

Two exercises, both on 2026-08-27, both recorded in full in the source repository.

**A live plugin.** An agent given only this skill built a working plugin from scratch and got
`BUILD SUCCESSFUL` and one passing test. Verification did not start green: the **first
`verifyPlugin` run failed**, the verifier rejecting the descriptor under two Marketplace rules
(`ForbiddenPluginIdPrefix` for the `com.example` prefix, `TemplateWordInPluginName` for the word
"plugin" in the name). `Compatible` came only after the descriptor was corrected — and that
correction, like the local verifier setup it ran under and the diagnosis of the disk incidents,
was the work of agents holding the full task context, not of the skill-only one. Targets: platform
dependency `intellijIdea("2025.2.6.2")`, which resolved to `IU-252.28539.54`, verified against a
locally installed IntelliJ IDEA 2026.2.1 (`IU-262.9437.185`); PHP claims checked against an
installed PhpStorm (`PS-262.9437.196`); platform sources read from an `intellij-community`
checkout on branch 262. It exposed four workflow defects, all fixed: the undeclared ~6.5 GB first
Gradle sync, `verifyPlugin` silently downloading a second IDE distribution into its own cache, no
documented way at all to pin the verifier's IDEs offline — the working DSL form had to be
recovered from the Gradle plugin's own class files with `javap` — and a `plugin.xml` example that
failed the very verifier rules the skill teaches.

**Seven acceptance scenarios.** Each was run by a fresh agent knowing only this skill, and judged
by a second agent that re-checked the artefacts itself — reading the produced code, the platform
sources, the compiled jars and the test XML — rather than believing the first agent's report.

All seven ended in PASS, and the route there is the honest part: scenarios 6 (compatibility) and
7 (bug fixing) **failed on the first run** and passed only after the skill was changed; scenario
2's first run was void through a fixture that already contained what the request asked for, and
was rerun on a stripped copy; the worst defect of all — the `ReadAction.computeBlocking` error
above — came out of scenario 3. Three fix passes were needed. Two further candidate defects were
investigated and **did not survive verification**, so nothing was written about them: an
intermittent `instrumentCode` failure that did not reproduce in 13 forced rebuilds, and a claimed
confusion between adding a `Disposable` and switching message buses that the judge could not find
in the files. Seven green ticks did not come for free, and this skill should not be read as
though they did.

**Validated on Claude only.** The Codex adapter (`agents/openai.yaml`) is shipped and the
frontmatter declares `targets: [claude, codex]`, but the skill has never been exercised on Codex.
Claiming a tested target that was not tested is the same error as claiming plugin compatibility
without running the verifier. Junie and JetBrains AI Assistant read these files as ordinary
Markdown; no adapter is shipped for them and none was tested.

The acceptance exercise also has stated limits, kept here rather than in a drawer: a "fresh
session" was a fresh subagent on the same machine with warm caches, not an isolated environment;
both reruns happened after the skill had been fixed, which tests the fix and not the original;
and 20 of the 45 `verify:` commands still stay green when their own file's body is emptied — they
re-derive a platform fact but assert nothing about the prose citing it. The repository's validator
now runs every `verify:` and enforces that gap as a closed list rather than leaving it unmeasured.

## Known deviations from the specification

The specification this skill was built from was not amended during execution. Where the shipped
skill differs, the skill is what shipped:

- **Antipatterns are split by topic, into seven files, not the two the specification prescribed**
  (with its own id assignment). The split follows subject matter, so a file's name matches its
  contents.
- **Reference count.** Every planning artefact in this project says 36 references. The shipped
  count is 45; it was measured, not copied, and the planning number was already stale four tasks
  before release.
- **One navigation row is worded differently from the plan text** — the row for
  `editor-inspections-completion.md` names inlay hints, because the file covers them and a reader
  searching for them would otherwise find no row.
- **The PhpStorm references locate the IDE through `PHPSTORM_HOME`**, not through the absolute
  path the plan used, so that no machine-specific path is shipped.

## What this skill is not

It is responsible for **knowledge of the platform**, not for understanding **your repository**.
Repository intelligence — JetBrains Context and equivalents — answers the second question, and
this skill neither duplicates it nor substitutes for it. It will not tell you why your codebase is
laid out as it is, which of your modules owns a feature, or what your team's conventions are; it
tells you what the platform actually offers and how to check that for yourself. It is also not a
replacement for reading your project's own code, and not a build system: it never asserts a Gradle
task name it has not listed from the project first.

Two things it deliberately refuses to do: claim compatibility that the plugin verifier has not
confirmed, and publish anything without your explicit confirmation of that specific publication.

## Reporting a problem

Open an issue at <https://github.com/Prihod/jetbrains-plugin-development-skill/issues>.

The most valuable report is a claim that turned out to be false at some build: name the file, the
symbol, the IDE and build number you targeted, and the command whose output disagrees with the
skill. That is exactly the failure this skill exists to prevent, and one was already found this
way during its own acceptance. Reference files carry a `verify:` line for this purpose — running
it and pasting the failure is a complete bug report.

## License

Apache License 2.0 — see `LICENSE`. This matches the sources this skill reads from and quotes:
`intellij-sdk-code-samples` ships plain Apache 2.0, and the `intellij-community` source files
carry Apache 2.0 headers. Note that `intellij-community`'s root `LICENSE.txt` is *not* Apache 2.0
but the JetBrains Open-Source Build Terms, which govern the distributed IDE build and themselves
state that it consists of open-source software subject to the Apache 2.0 license; the per-file
headers are what apply to the sources this skill quotes.

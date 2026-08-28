---
title: Verify, sign and publish a release
tags: workflow, release, verifier, signing, publishing
verify: PLUGIN_TEMPLATE_HOME="${PLUGIN_TEMPLATE_HOME:?}"; file=references/workflow-release-plugin.md; norm() { tr '[:space:]' ' ' | tr -s ' ' | sed -E 's/^ *([0-9]+\.|[-*]) *//' | sed -E 's/^ +//; s/ +$//'; }; pre_section=$(awk '/^### Preconditions$/{f=1;next}/^### /{f=0}f' "$file"); pre_block=$(printf '%s\n' "$pre_section" | awk '!inbul && /^[-*] \*\*Explicit/ { print; inbul=1; next } inbul && /^[-*] / { exit } inbul { print }'); steps_raw=$(awk '/^### Steps$/{f=1;next}/^### /{f=0}f' "$file"); publish_block=$(printf '%s\n' "$steps_raw" | awk '!instep && /^[0-9]+\. \*\*Publish\*\*/ { print; instep=1; next } instep && /^[0-9]+\. / { exit } instep { print }'); combined="$(printf '%s' "$pre_block" | norm)"$'\x1e'"$(printf '%s' "$publish_block" | norm)"; digest=$(printf '%s' "$combined" | shasum -a 256 | cut -c1-16); test "$digest" = "efeac2511cc4261c" && grep -qw -- 'PUBLISH_TOKEN' "$PLUGIN_TEMPLATE_HOME/.github/workflows/release.yml" && grep -qw -- 'publishPlugin' "$PLUGIN_TEMPLATE_HOME/.github/workflows/release.yml"
---

## Verify, sign and publish a release

Six steps take a finished change to a published Marketplace update. The last one is
irreversible and outward-facing — nothing in this file authorizes running it without
the user's explicit, in-the-moment confirmation for this specific release.

### Preconditions

- A plugin project whose build, tests and any intended platform upgrade are already done — see [workflow-create-plugin.md](workflow-create-plugin.md), [workflow-add-feature.md](workflow-add-feature.md), [workflow-upgrade-platform.md](workflow-upgrade-platform.md).
- Signing and publishing secrets available in the environment the publish step will run in — `PUBLISH_TOKEN`, `CERTIFICATE_CHAIN`, `PRIVATE_KEY`, `PRIVATE_KEY_PASSWORD` (`$PLUGIN_TEMPLATE_HOME/.github/workflows/release.yml`'s Publish Plugin step names exactly these).
- **Explicit user confirmation to publish this specific version**, obtained before the publish step runs — never assumed from "the build is green" or an earlier, more general go-ahead. Publishing is irreversible and outward-facing (`SKILL.md`'s Critical rule 12).

### Steps

1. **Reconcile version and changelog** — `gradle.properties`'s `version` and `CHANGELOG.md`'s `[Unreleased]` section describe the same release; read the pending entries with `./gradlew getChangelog --unreleased --no-header` (flags confirmed via `./gradlew help --task getChangelog`) before deciding the version is ready.
2. **Build** — the build task `./gradlew tasks` lists for this project; use the name it actually printed, not one recalled from another plugin.
3. **Test** — the test task from the same listing; do not proceed to signing on a build that only compiled.
4. **Verify across the whole declared range** — the verifier task from the same listing (`verifyPlugin` in the current template); see [compat-range-and-verifier.md](compat-range-and-verifier.md). A range in `plugin.xml` this step never ran against is not a verified range.
5. **Sign** — the signing task from the same listing (`signPlugin` in the current template); needs `CERTIFICATE_CHAIN`, `PRIVATE_KEY`, `PRIVATE_KEY_PASSWORD` from the Preconditions above.
6. **Publish** — the publish task from the same listing (`publishPlugin` in the current template); needs `PUBLISH_TOKEN` in addition to the signing secrets. **Do not run this step without the user's explicit confirmation, obtained for this specific version, immediately before running it.**

### Validation

Never assume Gradle task names. List what this project actually offers first:

```
./gradlew tasks
```

Steps 2–5 must all pass before Step 6 runs; a verifier or signing failure blocks publishing — it is fixed, never bypassed to keep a release on schedule.

### Common mistakes

A declared compatibility range the verifier was never actually run against — Step 4 skipped, or run against a narrower range than `plugin.xml` claims. Disabling or downgrading verification just to get a green build before a release, instead of fixing what it flags. Treating Step 6 as implied by Steps 2–5 passing — it needs its own, separate confirmation every time.

### References

[compat-range-and-verifier.md](compat-range-and-verifier.md); [workflow-upgrade-platform.md](workflow-upgrade-platform.md); [workflow-create-plugin.md](workflow-create-plugin.md); `SKILL.md`'s Critical rule 12; `$PLUGIN_TEMPLATE_HOME/.github/workflows/release.yml`, `CHANGELOG.md`, `gradle.properties`.

This file's `verify` pins a SHA-256 digest of the Preconditions confirmation
bullet and the Publish step above — the one irreversible, outward-facing
requirement in this skill — so a paraphrase cannot quietly weaken either one.
Editing either block is expected to fail `verify`; recompute the digest by
copying this file's `verify` line, deleting everything from `test "$digest"`
onward, appending `echo "$digest"` in its place, and running the result — then
paste the printed 16-character value back in as the new pinned digest.

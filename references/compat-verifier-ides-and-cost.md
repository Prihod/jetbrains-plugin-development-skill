---
title: Bind the verifier's IDE and know what it downloads
tags: plugin-verifier, ides, local, downloads, offline
verify: HOME="${HOME:?}"; f=references/compat-verifier-ides-and-cost.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); norm=$(printf '%s' "$body" | tr '\n' ' ' | tr -s ' '); printf '%s' "$norm" | grep -qF 'pluginVerification { ides { local(' || exit 1; for s in 'downloaded: 0 B' 'space used' 'first-run' 'unpacked' 'until-build' 'build/reports/pluginVerifier/<build>/report.html'; do printf '%s' "$norm" | grep -qF "$s" || exit 1; done; nums=$(printf '%s' "$norm" | sed -E 's/[A-Z]{2}-[0-9]+\.[0-9]+\.[0-9]+//g' | grep -oE '[0-9]+\.[0-9]+' | sort -u | sort -n); test "$(printf '%s\n' "$nums" | wc -l | tr -d ' ')" = 3 || exit 1; a=$(printf '%s\n' "$nums" | sed -n 1p); b=$(printf '%s\n' "$nums" | sed -n 2p); c=$(printf '%s\n' "$nums" | sed -n 3p); awk -v a="$a" -v b="$b" -v c="$c" 'BEGIN{d=a+b-c; if(d<0)d=-d; exit !(d<0.005 && b>=100*a)}' || exit 1; LP="$HOME/.pluginVerifier/loaded-plugins"; if [ -d "$LP" ]; then sz=$(du -sh "$LP" | awk '{print $1}'); printf '%s' "$norm" | grep -qF "$sz" || exit 1; mb=$(du -sm "$LP" | awk '{print $1}'); awk -v m="$mb" -v c="$c" 'BEGIN{d=m-c; if(d<0)d=-d; exit !(d<=2)}' || exit 1; fi; JAR=$(find "$HOME/.gradle/caches/modules-2" -name 'intellij-platform-gradle-plugin-*.jar' | head -1); test -n "$JAR" || exit 1; javap -classpath "$JAR" -p 'org.jetbrains.intellij.platform.gradle.extensions.IntelliJPlatformExtension$PluginVerification$Ides' | grep -q ' local(java.lang.String)' || exit 1; lscmd=$(printf '%s\n' "$body" | grep -F 'ls -d ~/.gradle/caches' | head -1); test -n "$lscmd" || exit 1; DIST=$(eval "${lscmd%%#*}" 2>/dev/null | head -1); test -n "$DIST" && test -d "$DIST/lib" || exit 1; bt=$(find "$DIST" -maxdepth 2 -name build.txt | head -1); test -n "$bt" || exit 1; real=$(cat "$bt"); btcmd=$(printf '%s\n' "$body" | grep -F 'maxdepth 2 -name build.txt' | head -1); test "$(eval "${btcmd%%#*}" 2>/dev/null)" = "$real" || exit 1; printf '%s\n' "$body" | grep -qF "$real" || exit 1; for x in $(printf '%s\n' "$body" | grep -oE '[A-Z]{2}-[0-9]+\.[0-9]+\.[0-9]+'); do test "$x" = "$real" || exit 1; done
---

## Bind the verifier's IDE and know what it downloads

`verifyPlugin` with no `ides { }` binding downloads its own IDE distribution(s) — separate from `intellijIdea(...)`'s,
into the Plugin Verifier's own home rather than the Gradle cache, possibly several if `recommended()`/`select()` matches
multiple releases (observed: `df -h /` fell 22Gi→16Gi in 90s on a bare probe; the daemon had to be killed). On this
machine that home is `~/.pluginVerifier/` with subdirectories `ides`, `extracted-plugins` and `loaded-plugins`; check
yours with `du -sh ~/.pluginVerifier/*` before and after a run rather than assuming the path. Bind an installed IDE
instead:

```kotlin
intellijPlatform {
    pluginVerification { ides { local("/path/to/Installed IDE.app") } }  // or File/Directory/Provider<String>
}
```

`--list-ides` confirms the binding with no download; `downloaded: 0 B` in the run log confirms the **IDE itself** stayed
local. What `local()` does not cover is the verifier's separate cache of the bundled plugins it pulls in as dependencies
of the IDE under test — and that is a **first-run** cost, not a per-run one. Measured on this Baseline, same plugin and
same bound IDE: `downloaded: 979.36 MB`, then `5.65 MB`, then `0 B` on every run since.

Read the verifier's last two lines as two different quantities. `Total amount of plugins and dependencies downloaded` is
this run's network transfer; `Total amount of space used for plugins and dependencies` is the accumulated cache. Here
they read `0 B` and `985.01 MB` in the same run — 985.01 is exactly 979.36 + 5.65, and `du -sh
~/.pluginVerifier/loaded-plugins` reports the same 985M — so a large "space used" beside `downloaded: 0 B` costs
nothing. Budget the fetch once per bound IDE, then treat the task as offline-capable.

`--offline` really does force it: the same plugin and IDE verified `Compatible` in 8 s with `space used: 0 B`, against
56 s and 985.01 MB used online. That is not a cheaper equivalent — the dependency plugins are never loaded — so use it
to prove no network is needed, not as the run you rely on.

This DSL is absent from `$PLUGIN_TEMPLATE_HOME`, its CI workflow, and `./gradlew help --task verifyPlugin` (CLI flags
only) — when template and `help` both miss a Gradle-plugin block, read its signature from the resolved jar rather than
guess:

```bash
JAR=$(find ~/.gradle/caches/modules-2 -name 'intellij-platform-gradle-plugin-*.jar' | head -1)
javap -classpath "$JAR" -p 'org.jetbrains.intellij.platform.gradle.extensions.IntelliJPlatformExtension$PluginVerification$Ides'
```

JVM signatures from the plugin's own compiled classes, not decompiled platform source (`source-lookup.md`'s Gradle-cache
warning covers missing platform `-sources.jar`, a different case) — confirmed `local(String|File|Directory|Provider<?>)`
alongside `current()`, `latest(...)`, `recommended()`, `select(...)`.

## The floor build is already unpacked on disk

The cheapest IDE to bind is the one the project already compiles against. The resolved `intellijIdea(...)` /
`intellijIdeaCommunity(...)` dependency is not kept as an archive: Gradle's transform cache leaves it unpacked as a
complete IDE tree — on macOS, the top-level entries an installed `IntelliJ IDEA.app/Contents` has (`bin jbr lib
modules plugins help license Resources ...`) — and `local()` takes that directory as-is, so nothing downloads.
Locate it and read which build it is as in [source-lookup-target-build.md](source-lookup-target-build.md); never
paste the hashed path into a build file — it changes with the pinned version and differs per machine:

```bash
ls -d ~/.gradle/caches/*/transforms/*/transformed/idea*/   # one line per unpacked platform dependency
DIST=<the line whose version matches the pin in build.gradle.kts>
find "$DIST" -maxdepth 2 -name build.txt -exec cat {} \;   # which build it is, e.g. IU-252.28539.54
```

Unless `sinceBuild` was widened past the compiled-against platform, that build **is** the floor of the declared
range, so binding it verifies the floor at zero download cost — the first rung of the ladder in
[compat-range-and-verifier.md](compat-range-and-verifier.md), ahead of every rung that costs a fetch. Pass the path
in from a Gradle property or environment variable (`local()` also accepts `Provider<String>`, above) instead of
committing a machine-specific literal.

**Verifying forward is the same mechanism.** An open `until-build` claims every future release, and no finite number
of runs discharges that claim — but "not provable in full" is no reason to leave the open end untouched. Add a
second `local(...)` for the newest IDE actually installed here, and a single `verifyPlugin` run covers both ends:
one `build/reports/pluginVerifier/<build>/report.html` is written per bound IDE, named for the build it read, so
the floor and the practical ceiling become two reports to open rather than two assumptions.

Reference: [compat-range-and-verifier.md](compat-range-and-verifier.md);
`$PLUGIN_TEMPLATE_HOME/.run/Run Verifications.run.xml`, `.github/workflows/build.yml`;
the resolved `intellij-platform-gradle-plugin-<version>.jar` under `~/.gradle/caches/modules-2`;
`~/.pluginVerifier/loaded-plugins` and the `verifyPlugin` task's own log.

---
title: Touch project files through the VFS, not java.io.File
tags: vfs, virtual-file, refresh, listeners
verify: IJ_SRC="${IJ_SRC:?}"; f=references/model-vfs.md; body=$(awk 'BEGIN{c=0}/^---$/{c++;next}c>=2' "$f"); printf '%s\n' "$body" | grep -qF 'UserDataHolderBase' || exit 1; printf '%s\n' "$body" | grep -qF 'ModificationTracker' || exit 1; grep -n "public abstract class VirtualFile extends UserDataHolderBase implements ModificationTracker" "$IJ_SRC/platform/core-api/src/com/intellij/openapi/vfs/VirtualFile.java"
---

## Touch project files through the VFS, not java.io.File

`VirtualFile`
(`platform/core-api/src/com/intellij/openapi/vfs/VirtualFile.java:58`, an abstract
class extending `UserDataHolderBase` and implementing `ModificationTracker`) is the
IDE's model of a file: it backs PSI, indices, and the editor. `java.io.File`/`java.nio`
know nothing about that model.

**Wrong (AP-11 — the IDE's model doesn't see this write):**

```kotlin
File(path).writeText(newContent) // editors and indices keep showing stale content
```

**Right — go through the VFS so the model stays consistent:**

```kotlin
VfsUtil.saveText(virtualFile, newContent) // platform/analysis-api .../VfsUtil.java:49
```

When a file is created or changed by a tool outside the VFS's view (an external build,
a spawned process, `ProcessBuilder`), refresh afterwards so the model catches up:
`virtualFile.refresh(asynchronous, recursive, postRunnable)`, or
`LocalFileSystem.getInstance().refreshIoFiles(files)` for a batch of `java.io.File`s
you don't yet hold a `VirtualFile` for.

**Listening for changes.** Subscribe to `VirtualFileManager.VFS_CHANGES`
(`platform/core-api/.../VirtualFileManager.java:36`, a `Topic<BulkFileListener>`) on
the message bus — `project.getMessageBus().connect(disposable).subscribe(VFS_CHANGES,
listener)` — rather than polling. `BulkFileListener`
(`platform/core-api/src/com/intellij/openapi/vfs/newvfs/BulkFileListener.java:38`)
delivers batched `VFileEvent`s after changes are applied; `AsyncFileListener`
(`platform/core-api/.../AsyncFileListener.java`) is the variant for computing the
listener's reaction off the EDT before a synchronous apply. Both need a `Disposable`
owner for the subscription — see
[lifecycle-disposable-messagebus.md](lifecycle-disposable-messagebus.md).

**When `java.io.File` is fine.** Files outside the project's content roots entirely —
a temp file the plugin creates and deletes itself, a config file the IDE's model has no
reason to track, reading a bundled resource off the classpath. The rule is about files
the *project* considers part of itself; once a file leaves that scope, there's no
IDE-side model to go stale.

Reference: `platform/core-api/src/com/intellij/openapi/vfs/VirtualFile.java`;
`platform/analysis-api/src/com/intellij/openapi/vfs/VfsUtil.java` (`saveText`);
`platform/analysis-api/src/com/intellij/openapi/vfs/LocalFileSystem.java`
(`refreshIoFiles`); `platform/core-api/src/com/intellij/openapi/vfs/VirtualFileManager.java`
(`VFS_CHANGES`); `platform/core-api/src/com/intellij/openapi/vfs/newvfs/BulkFileListener.java`;
`platform/core-api/src/com/intellij/openapi/vfs/AsyncFileListener.java`;
`platform/core-api/src/com/intellij/openapi/vfs/newvfs/events/VFileEvent.java`.

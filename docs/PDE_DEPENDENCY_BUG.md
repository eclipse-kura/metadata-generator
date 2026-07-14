# `vscode-pde` broken by upstream `redhat.java` (jdt.ls) update

## Summary

As of late June 2026, the [`yaozheng.vscode-pde`](https://marketplace.visualstudio.com/items?itemName=yaozheng.vscode-pde)
extension (Eclipse PDE support for VS Code) is completely broken when paired with current
versions of [`redhat.java`](https://marketplace.visualstudio.com/items?itemName=redhat.java)
(Language Support for Java). PDE-nature projects (OSGi bundles using
`org.eclipse.pde.core.requiredPlugins` in their `.classpath`) fail to resolve their
dependencies, showing up in VS Code as widespread "Java dependencies failing" / unresolved
import errors across an entire Kura/PDE workspace.

This is **not** caused by workspace location, project configuration, Maven state, or any
per-project setting — it reproduces on any machine/workspace using these two extension
versions together. It also is **not** a `vscode-pde` regression by itself — `vscode-pde` has
been unchanged for two years. It was broken by a dependency-removal change in the upstream
Eclipse JDT Language Server (jdt.ls), which `redhat.java` bundles.

## Background

`vscode-pde` doesn't run standalone — it contributes a set of jars to `redhat.java`'s
`javaExtensions` mechanism, which are loaded directly into the running jdt.ls OSGi (Equinox)
container. This lets it add Eclipse PDE plugin classes (`org.eclipse.pde.core`,
`org.eclipse.pde.launching`, `org.eclipse.m2e.pde.target`, `org.eclipse.jdt.ls.importer.pde`,
etc.) into jdt.ls's own runtime, so jdt.ls can import and resolve PDE/OSGi bundle projects
against a target platform.

Those PDE jars bundled by `vscode-pde` include Eclipse's `org.eclipse.e4.core.contexts`
(v1.12.500), which **mandatorily imports** the OSGi Compendium package:

```
Import-Package: org.osgi.service.event;version="[1.3.0,2.0.0)"
```

`vscode-pde` does **not** ship anything that exports this package itself. Historically, this
was never a problem, because `redhat.java`'s own bundled jdt.ls shipped the
`org.eclipse.osgi.services` bundle (which exports `org.osgi.service.event`) as part of its own
dependency set — `vscode-pde` was implicitly relying on it being present.

## The breaking change

Upstream jdt.ls removed that bundle, believing it to be unused by jdt.ls itself (which is
true — jdt.ls's own core doesn't need it):

- **PR:** [eclipse-jdtls/eclipse.jdt.ls#3826 — "Remove `org.eclipse.osgi.services` from category.xml"](https://github.com/eclipse-jdtls/eclipse.jdt.ls/pull/3826)
- **Merge commit:** `1ca9d6ac493c64286c2722d8abc396ab2a4dcd60`
- **Merged:** 2026-06-26

jdt.ls has no visibility into third-party `javaExtensions` contributors like `vscode-pde`, so
there was no way for them to know this bundle was a silent, load-bearing dependency for the
PDE extension. Any `redhat.java` build that picked up this jdt.ls change breaks `vscode-pde`.

## Effect on PDE

With `org.osgi.service.event` unresolved, the OSGi resolver fails a cascading chain of
bundles, all traceable in the jdt.ls log
(`<workspaceStorage>/redhat.java/jdt_ws/.metadata/.log`):

```
org.osgi.service.event                (missing — nothing exports it)
  → org.eclipse.e4.core.contexts       fails to resolve
    → org.eclipse.e4.core.services     fails
      → org.eclipse.pde.core           fails
      → org.eclipse.pde.launching      fails
        → org.eclipse.m2e.pde.target   fails
          → org.eclipse.jdt.ls.importer.pde   fails  ← this is what actually
                                                          imports PDE bundle projects
```

`org.eclipse.jdt.ls.importer.pde` failing to start means jdt.ls can no longer resolve
`org.eclipse.pde.core.requiredPlugins` classpath entries. Practically, every PDE-nature
project in the workspace shows unresolved-dependency errors on anything coming from the
target platform, even though the project files themselves are completely correct.

## How to confirm you're hitting this

Open the "Language Support for Java" output channel, or the jdt.ls log
(`<workspaceStorage>/redhat.java/jdt_ws/.metadata/.log`), and look for:

```
!MESSAGE Bundle startup failed reference:file:.../yaozheng.vscode-pde-<version>/server/org.eclipse.e4.core.services_....jar
org.osgi.framework.BundleException: Could not resolve module: org.eclipse.e4.core.services [...]
  Unresolved requirement: Require-Bundle: org.eclipse.e4.core.contexts
    ...
       Unresolved requirement: Import-Package: org.osgi.service.event; version="[1.3.0,2.0.0)"
```

## Fixes

There are two independent ways to unblock this. **Fix B (add the missing bundle) is
recommended** — it doesn't require freezing `redhat.java` at an old version, and it's a
one-time, low-risk local change.

### Fix A — Downgrade `redhat.java` (not recommended, keep as fallback only)

Downgrade the "Language Support for Java" extension to a build that predates jdt.ls PR #3826
(e.g. `1.54.0`, published ~2026-04-11). This avoids the removed bundle entirely, but:
- You lose subsequent jdt.ls improvements/fixes.
- Extension auto-update will silently re-break this unless disabled for `redhat.java`.
- It's a workaround for a problem that isn't really `redhat.java`'s fault — jdt.ls itself is
  fine; it's `vscode-pde`'s hidden dependency that's the real gap.

### Fix B — Add the missing bundle to `vscode-pde` (recommended)

Manually give `vscode-pde` the one jar it's missing, so it stops depending on `redhat.java` to
provide it. This survives future `redhat.java` updates and only needs to be redone if
`vscode-pde` itself is reinstalled/updated.

#### ⚠️ Which jar version to use, and why

Use **`org.eclipse.osgi.services` version `3.10.200`** from Maven Central. Do **not** grab the
latest version (3.12.x) — as of that release, Eclipse split `org.eclipse.osgi.services` into
several separate re-exporting bundles (it now just does
`Require-Bundle: org.osgi.service.event; ...; visibility:=reexport` instead of exporting the
package itself), which means adding the newest jar alone doesn't fix anything — you'd need to
also add every bundle it now re-exports. Version `3.10.200` is the last self-contained
release: it directly exports `org.osgi.service.event` (version 1.4, satisfying the required
`[1.3.0,2.0.0)` range) with no further missing dependencies. It has no other unresolved
`Require-Bundle` entries either — it's a clean drop-in.

Maven Central coordinates: `org.eclipse.platform:org.eclipse.osgi.services:3.10.200`
Direct URL:
```
https://repo1.maven.org/maven2/org/eclipse/platform/org.eclipse.osgi.services/3.10.200/org.eclipse.osgi.services-3.10.200.jar
```

#### Step by step

**1. Locate your `vscode-pde` extension folder.**

The path depends on whether VS Code is running locally or connected to a remote host (e.g.
Remote-SSH into a Debian/Linux dev box) — extensions for a *remote* connection live on the
remote machine, not your laptop, regardless of whether your laptop is macOS or Linux.

| Setup | Extension folder |
|---|---|
| VS Code Remote-SSH (client is macOS **or** Linux/Debian, connecting to a remote Linux host) | `~/.vscode-server/extensions/yaozheng.vscode-pde-<version>/` **on the remote host** |
| VS Code running directly on Debian/Linux (no remote connection) | `~/.vscode/extensions/yaozheng.vscode-pde-<version>/` |
| VS Code running directly on macOS (no remote connection) | `~/.vscode/extensions/yaozheng.vscode-pde-<version>/` |

(macOS and Linux desktop installs both use `~/.vscode/extensions` — there's no
`Application Support` path involved for extensions specifically.)

The `<version>` at time of writing is `0.11.1` — the only version ever published, last
updated 2024-07-31.

**2. Download the jar into that extension's `server/` folder.**

```bash
cd ~/.vscode-server/extensions/yaozheng.vscode-pde-0.11.1/server   # or ~/.vscode/extensions/... if not using Remote-SSH

curl -sL -o org.eclipse.osgi.services_3.10.200.jar \
  "https://repo1.maven.org/maven2/org/eclipse/platform/org.eclipse.osgi.services/3.10.200/org.eclipse.osgi.services-3.10.200.jar"
```

This command is identical on Debian and macOS.

**3. Register the jar in `package.json`.**

Open `~/.vscode-server/extensions/yaozheng.vscode-pde-0.11.1/package.json` (adjust the base
path per the table above) in a text editor. Find the `contributes.javaExtensions` array — it
ends with a list of `./server/...jar` paths. Add the new jar as one more entry, being careful
to keep the JSON valid (add a comma after what was previously the last entry):

```json
			"./server/org.apache.commons.commons-io_2.16.1.jar",
			"./server/javax.annotation_1.3.5.v20200504-1837.jar",
			"./server/javax.inject_1.0.0.v20091030.jar",
			"./server/org.eclipse.osgi.services_3.10.200.jar"
		],
```

A JSON syntax error here (missing/extra comma, mismatched bracket) will make the whole
extension fail to load — double-check the edit before saving.

**4. Fully restart the Java language server / extension host.**

A simple "Reload Window" is not always sufficient in Remote-SSH setups — the previous jdt.ls
process can linger and cause unrelated duplicate-bundle-install errors on the next start. To
be safe: close the remote connection entirely (or kill the VS Code Server process on the
remote host) and reconnect, rather than just reloading the window.

**5. Verify.**

Reopen the workspace and check the jdt.ls log
(`<workspaceStorage>/redhat.java/jdt_ws/.metadata/.log`) for the `BundleException` /
`Could not resolve module` chain described above. It should no longer appear, and PDE
projects should resolve their dependencies normally.

## Ask for `vscode-pde` upstream

The root gap is that `vscode-pde` has an undeclared, implicit dependency on
`org.eclipse.osgi.services` being provided by the host (`redhat.java`/jdt.ls), which is no
longer a safe assumption. The extension should bundle its own copy of a compatible
`org.eclipse.osgi.services` (or an equivalent bundle exporting `org.osgi.service.event`)
directly, the same way it already bundles its other PDE/Equinox dependencies, rather than
relying on jdt.ls to supply it.

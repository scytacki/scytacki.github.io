---
layout: page
title: "Using prerelease dependency versions when a peerDependency excludes them"
description: "Why a prerelease that a transitive peerDependency can't satisfy gets installed twice, and how npm, Yarn, and pnpm — and their override/peer settings — differ."
permalink: /prerelease-peer-dependency-package-managers/
---

A reference for what happens when you try to use a **prerelease** version of a library that some *other* package declares as a `peerDependency` with a stable range — and how the major package managers, under their various settings, each resolve it.

## Goal driving this

**Goal driving this:** make it cheap for a developer to push code in a dependent package against a **prerelease** of a shared library, have CI test it, and ship it for manual QA — *before* the library's final release. Problems are usually found only once a dependent package actually consumes the new version, so you want the prerelease in the real dependency graph with as little friction as possible. The friction comes from how package managers handle a prerelease that a transitive `peerDependency` range can't satisfy.

## The scenario

Three packages, generic names throughout:

- **`engine`** — the shared library. Latest stable is `engine@3.4.0`; you want to test the prerelease `engine@3.5.0-pre.0`.
- **`plugin`** — depends on `engine` as a **peer**: `"peerDependencies": { "engine": ">=3.0.0" }`.
- **`app`** — your project (or a workspace package). It depends on `plugin`, and you bump its own dependency to `"engine": "3.5.0-pre.0"` to test the prerelease.

You expect one copy of `engine` (the prerelease) to serve everyone. Whether that happens depends entirely on the package manager.

## Why a second copy appears

Two facts combine:

1. **Semver excludes prereleases from ranges that don't name one.** `3.5.0-pre.0` does **not** satisfy `>=3.0.0`. Per the [node-semver prerelease rule](https://github.com/npm/node-semver#prerelease-tags), a version with a prerelease tag only satisfies a comparator when that comparator shares its exact `[major, minor, patch]` tuple **and** itself carries a prerelease tag. `>=3.0.0` has no prerelease tag, so the prerelease is invisible to it.
2. **Some package managers auto-install peers.** When the manager sees `plugin`'s unmet `engine` peer and notices the hoisted prerelease can't satisfy it, it may install a *second*, stable `engine@3.4.0` to fill the peer slot.

The result is two physical copies of `engine` in one tree. That is harmless for many libraries but breaks any library that:

- runs a **module-level singleton guard** that throws when loaded twice (common in SDKs that talk to a host frame/process),
- relies on `instanceof` / shared module state / a registry across the boundary, or
- is something like **React**, where two copies produce "invalid hook call" and broken context.

It also silently inflates bundles. The whole question below is: *which managers create the second copy, and what setting collapses it back to one?*

## Behavior by package manager

Default behavior with the scenario above (`app` directly depends on `engine@3.5.0-pre.0`; `plugin` peer-needs `engine >=3.0.0`):

| Package manager | Auto-installs a 2nd stable copy? | Default result | Exit |
| --- | --- | --- | --- |
| **npm** 7–11 | **yes** | **2 copies** (`3.4.0` for the peer + `3.5.0-pre.0`) | 0, with peer notices |
| **Yarn 1** (classic) | no | **1 copy** (`3.5.0-pre.0`) | 0, unmet-peer warning |
| **Yarn Berry** (node-modules linker) | no | **1 copy** | 0, peer warning |
| **Yarn Berry** (default PnP linker) | no | keeps whatever versions are *requested*; peer left **unprovided** | 0 at install; strict at **runtime** |
| **pnpm** | no — binds the peer to the available version | **1 copy** | 0, peer notice |

**The takeaway: the duplicate is npm-specific.** Yarn and pnpm satisfy `plugin`'s peer from the prerelease that is already present and merely *warn* that it's outside the declared range. npm instead treats the unmet range as something to *fix* by installing another copy.

A useful corollary: a hard `ERESOLVE`/peer **error** during a clean install is also mostly an npm trait. Yarn and pnpm downgrade peer conflicts to warnings by default, so an install that npm aborts will often complete (with warnings) under them.

## Settings that force a single copy

Each manager has a knob. They differ in *what* they change and — importantly — in whether they apply on a normal install or require wiping the lockfile.

### npm — `overrides` (resolution rewrite)

```jsonc
// package.json
"overrides": { "engine": "3.5.0-pre.0" }
```

`overrides` is a **resolution-layer rewrite**: it substitutes the named version for *every* request of `engine` in the tree — direct, transitive, and **peer slots** — regardless of what each package declares. It does collapse the tree to a single copy. Caveats:

- **It only takes effect on a from-scratch resolution.** With an existing `package-lock.json`, a normal `npm install` reproduces the locked tree and never re-evaluates a newly-added override. You must delete `package-lock.json` (and usually `node_modules`) and reinstall. This is long-standing, documented behavior — see [npm/cli #4232](https://github.com/npm/cli/issues/4232) and the [package-locks docs](https://docs.npmjs.com/cli/v9/configuring-npm/package-lock-json). It is **not** workspace-specific; it reproduces in a single-package project.
- **`EOVERRIDE`:** you cannot override a package the *current* `package.json` directly depends on to a conflicting spec — npm errors `Override for engine@^3.0.0 conflicts with direct dependency`. In a workspaces monorepo the *root* usually doesn't depend on `engine` (only the workspace packages do), so a root override is free to rewrite everything; in a single package you must change the direct spec to match.
- Forcing a prerelease into a peer slot leaves `npm ls` reporting the peer as `invalid` (exit code `ELSPROBLEMS`), even though `install`/`ci`/`build` succeed. It's a warning, not a failure — but strict `npm ls` checks in CI will flag it.

### npm — `legacy-peer-deps` (stop auto-installing peers)

```ini
# .npmrc
legacy-peer-deps=true
```

This reverts npm to its pre-v7 behavior: peers are the consumer's responsibility and are **not** auto-installed — so no second stable copy is created. It reproduces Yarn 1's behavior while staying on npm. Two advantages over `overrides`:

- **It applies on a normal incremental `npm install`** — it will collapse an existing two-copy tree to one without deleting the lockfile, and `npm ci` reproduces it. (It changes how npm reifies the tree rather than requiring a fresh ideal-tree build, so the #4232 short-circuit doesn't apply.)
- No per-version edit: a developer just bumps their package's `engine` version and installs.

The cost: it's **repo-wide**. It silences *all* peer resolution, including genuinely useful peer checks, so real peer mismatches elsewhere stop being surfaced.

### Yarn 1 / Yarn Berry — `resolutions`

```jsonc
// package.json
"resolutions": { "engine": "3.5.0-pre.0" }
```

Forces a single version everywhere, like `overrides`. Because Yarn **reconciles the lockfile with `package.json` on every install**, a newly-added resolution (or a version bump) takes effect on a normal `yarn install` — no lockfile-delete dance. Note that `resolutions` fixes *versions*; under Yarn Berry's PnP it does **not** by itself make an ancestor *provide* an unprovided peer (see below).

### Yarn Berry — `packageExtensions`

```yaml
# .yarnrc.yml
packageExtensions:
  "plugin@*":
    dependencies:
      engine: "3.5.0-pre.0"
```

The most surgical option, and one npm lacks: it **rewrites a third-party package's manifest without forking it**. You can turn `plugin`'s `engine` peer into a real dependency, or widen/relabel it, so the peer is satisfied cleanly and the warning disappears entirely. Applies on a normal install.

### pnpm — `overrides`, `packageExtensions`, `peerDependencyRules`

```jsonc
// package.json
"pnpm": {
  "overrides": { "engine": "3.5.0-pre.0" },
  "packageExtensions": { "plugin": { "dependencies": { "engine": "3.5.0-pre.0" } } },
  "peerDependencyRules": { "allowedVersions": { "plugin>engine": "3.5.0-pre.0" } }
}
```

pnpm already produces a single copy by default here, so the settings are mostly about **accepting** the out-of-range peer rather than deduping. `peerDependencyRules.allowedVersions` tells pnpm to treat the prerelease as an allowed match for that peer (silencing the warning); `overrides` and `packageExtensions` mirror their Yarn/npm counterparts. Like Yarn, pnpm reconciles on every install, so these don't need a lockfile wipe.

### Why only npm needs the lockfile-delete dance

Yarn and pnpm recompute the resolution against `package.json` on **every** install, so a new resolution/override/version is picked up immediately. npm treats an existing `package-lock.json` as the source of truth to *reproduce*, and only re-derives overrides during a from-scratch resolve. That asymmetry — not anything about monorepos — is the entire developer-experience gap for the `overrides` path. (npm's `legacy-peer-deps` avoids it because it isn't an override at all.)

## Yarn Berry PnP, specifically

Berry's default linker is **Plug'n'Play** (no `node_modules`; a `.pnp.cjs` resolution map). Its behavior differs from the node-modules world:

- An install with an out-of-range or unprovided peer **does not hard-error** — it exits 0 with peer warnings (`YN0002` "X doesn't provide engine, requested by plugin"; `YN0086` "peer dependencies incorrectly met"). It keeps exactly the versions requested (so a mixed graph keeps multiple `engine` versions).
- PnP is **strict at runtime**: a package can only `require` what it (or a *provided* peer) declares. So an unprovided peer surfaces as a real resolution failure when code runs — not a silent fallback to a hoisted copy. This is stricter than npm/Yarn-classic, and it's why `packageExtensions` (which actually provides the peer) is the clean fix, while `resolutions` alone (which only fixes versions) leaves the provision gap.

## The `>=3.0.0-0` non-fix

A tempting "fix" is to loosen `plugin`'s peer to `>=3.0.0-0`, on the theory that the `-0` admits prereleases. **It does not generalize.** Under the semver prerelease rule, the `-0` only opens prereleases that share the comparator's exact `[major, minor, patch]` — i.e. `>=3.0.0-0` admits `3.0.0-pre.x` but still **excludes `3.5.0-pre.0`** (tuple `[3,5,0] ≠ [3,0,0]`). There is no dependency-range syntax meaning "allow any prerelease of any version"; that requires semver's `includePrerelease` *option*, which a `package.json` range can't express. So loosening the peer range is a dead end — the levers are the per-consumer settings above, or publishing a non-prerelease version (which satisfies the range and makes the whole problem disappear).

## Summary matrix

| Manager | Default copies | Knob for one copy | Applies on normal install? | Notes |
| --- | --- | --- | --- | --- |
| npm | 2 | `overrides` | **no** — needs lockfile delete | resolution rewrite; `EOVERRIDE` on own direct dep |
| npm | 2 | `legacy-peer-deps=true` | **yes** | repo-wide; silences all peer checks |
| Yarn 1 | 1 | `resolutions` (if versions diverge) | yes | no peer auto-install |
| Yarn Berry (nm) | 1 | `resolutions` / `packageExtensions` | yes | `packageExtensions` can rewrite the peer |
| Yarn Berry (PnP) | per request | `packageExtensions` | yes | strict at runtime; provide the peer |
| pnpm | 1 | `peerDependencyRules` / `overrides` / `packageExtensions` | yes | default dedup; rules accept the peer |

## How to reproduce

Create three packages matching the scenario (a `plugin` with `"peerDependencies": { "engine": ">=3.0.0" }`, and an `app` that depends on `plugin` and directly on `engine@<prerelease>`), then for each manager:

```bash
# npm — observe two copies, then collapse
npm install
find . -path '*/node_modules/engine/package.json' -exec node -p '`${require("./"+process.argv[1]).version}`' {} \;   # 2 versions
echo 'legacy-peer-deps=true' > .npmrc && npm install   # now 1 — applied incrementally
# (overrides instead would need: rm -rf node_modules package-lock.json && npm install)

# Yarn classic / Berry
yarn install            # 1 copy already; check with: yarn why engine

# pnpm
pnpm install            # 1 copy; check with: ls node_modules/.pnpm | grep engine@
```

Count **physical** copies, not lockfile text: a working npm `override` writes *no* `overrides` key visible to a naive `grep`, so diagnose by the number/version of installed copies (`find … node_modules/engine`, `yarn why`, or `.pnpm` store entries).

## References

- [node-semver — Prerelease Tags](https://github.com/npm/node-semver#prerelease-tags) — the rule that a prerelease only satisfies a comparator sharing its `[major,minor,patch]` and carrying a prerelease tag.
- [npm/cli #4232 — Overrides are not updating after running npm install](https://github.com/npm/cli/issues/4232) — overrides aren't re-evaluated against an existing lockfile; delete it to apply.
- [npm/cli #9358 — incomplete package-lock when overrides override peer deps](https://github.com/npm/cli/issues/9358) — an override that forces a version into a peer slot can produce a lockfile that later breaks `npm ci`; verify after baking it in.
- [npm docs — package-lock.json](https://docs.npmjs.com/cli/v9/configuring-npm/package-lock-json) and [overrides](https://docs.npmjs.com/cli/v9/configuring-npm/package-json#overrides).
- [npm docs — `legacy-peer-deps`](https://docs.npmjs.com/cli/v9/using-npm/config#legacy-peer-deps).
- [Yarn — `resolutions`](https://yarnpkg.com/configuration/manifest#resolutions) and [`packageExtensions`](https://yarnpkg.com/configuration/yarnrc#packageExtensions).
- [Yarn — peer dependency warnings / PnP](https://yarnpkg.com/advanced/lexicon#peer-dependency) — Berry treats peer issues as warnings; PnP is strict at runtime.
- [pnpm — `peerDependencyRules`](https://pnpm.io/package_json#pnpmpeerdependencyrules), [`overrides`](https://pnpm.io/package_json#pnpmoverrides), and [`packageExtensions`](https://pnpm.io/package_json#pnpmpackageextensions).

---
_Last updated: 2026-06-26_

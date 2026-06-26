---
layout: page
title: Reference notes
description: Reference notes on web development, accessibility, and related topics by Scott Cytacki.
---

Reference notes on web development, accessibility, and related topics. Suggestions and PRs welcome — the source lives at [github.com/scytacki/scytacki.github.io](https://github.com/scytacki/scytacki.github.io).

## Notes

- [`KeyboardEvent.key` with Shift + modifiers across platforms (US keyboards)](./keyboardevent-shift-modifiers-us/) — reference tables for how `event.key` behaves when Shift is combined with Cmd/Ctrl/Alt across macOS, Windows, and ChromeOS, plus how to reproduce them.
- [Working with keyboard shortcuts in browsers: cross-platform pitfalls](./keyboard-shortcuts-cross-platform/) — what that per-platform behavior means in practice: why matching on the typed character is unreliable, what to do instead, and how it maps to `aria-keyshortcuts`.
- [Using prerelease dependency versions when a peerDependency excludes them](./prerelease-peer-dependency-package-managers/) — why a prerelease that a transitive `peerDependency` can't satisfy gets installed twice, and how npm, Yarn, and pnpm — and their override/peer settings — differ.

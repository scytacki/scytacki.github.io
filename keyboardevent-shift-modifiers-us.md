---
layout: page
title: "KeyboardEvent.key with Shift + modifiers across platforms (US keyboards)"
description: "How event.key behaves when Shift is combined with Cmd/Ctrl/Alt across macOS, Windows, and ChromeOS, on US keyboards."
permalink: /keyboardevent-shift-modifiers-us/
---

A reference for how `event.key` behaves when **Shift is combined with a control/command/alt modifier** across platforms and browsers.

## Summary of inconsistent `event.key` values

Chrome 148 on MacOS `Ctrl+Shift+[key]`:
- the shifted character for letters.
- the unshifted character for symbols and digits.
- **except** `2`, `6`, `-`, `[`, `]`, and `\` report their shifted character.

Chrome 148 on MacOS `Cmd+Shift+[key]`: the unshifted character.

Safari and Firefox on MacOS:
- `Ctrl+Shift+[key]` reports the shifted character
- `Cmd+Shift+[key]` reports the unshifted character

Chrome on Windows and ChromeOS always reports the shifted character.

## Scope & assumptions

- **US keyboard layout only.** Other layouts reach these characters differently and will not match this table.
- Platforms tested: **macOS**, **Windows**, **ChromeOS (Chromebook)**.
- Assumed **Chromium-based browser** (Chrome/Edge), except where a table says otherwise. Firefox and Safari do differ for `Ctrl+Shift` — see the cross-browser `.` table below.
- MacOS Chrome version 148 – the behavior on MacOS has changed in the past and will likely changed in the future.
- **Linux is not a column.** Common desktop Linux (Chromium/X11/Wayland) is expected to track the **ChromeOS** column; verify if it matters to you.
- `event.code` is **not** in the table because it is layout- and modifier-independent: the physical key here is always `Slash` (for `/`), `Digit1`, `KeyA`, etc., on every platform.
- The keys pressed column uses the modifier key names from [UI Events Key spec](https://www.w3.org/TR/uievents-key/#keys-modifier). This means that Mac `Option` is listed as `Alt`, and Mac `Cmd` is listed as `Meta`.

## Chrome cross platform `/` key

`event.key` for different modifiers and platforms. All tested on Chrome. The Windows key was used for Meta. 

| Keys pressed   | macOS | Windows | ChromeOS |
| -------------- | ----- | ------- | -------- |
| `/`            | `/`   | `/`     | `/`      |
| `Shift+/`      | `?`   | `?`     | `?`      |
| `Ctrl+Shift+/` | `/`†  | `?`     | `?`      |
| `Alt+Shift+/`  | `¿`‡  | `?`     | `?`      |
| `Meta+Shift+/` | `/`†  | `?`     | n/a      |

### † Non glyph modifier combined with Shift
On macOS holding **either** `Meta(Cmd)` or `Ctrl` with `Shift+/` reports the **unshifted** `/`. 

### ‡ Glyph modifier combined with Shift
On a Mac the `Alt(Option)+Shift+/` actually types a character. Mac's `Option` is like the AltGraph modifier on other platforms. So the `event.key` of `Alt(Option)+Shift+/` matches what is typed.

> Note: this MacOS behavior is **not consistent** across all symbols and digits. See below.

## MacOS cross browser `.` key

`event.key` for the `.` key with different browsers. The `.` key is used because `Meta(Cmd)+shift+/` brings up help in the browsers and in some cases blocks the browser from receiving the event.

| Keys pressed     | Chrome | Firefox | Safari |
| ---------------- | ------ | ------- | ------ |
| `.`              | `.`    | `.`     | `.`    |
| `Shift+.`        | `>`    | `>`     | `>`    |
| `Ctrl+Shift+.`   | `.`†   | `>`‡    | `>`‡   |
| `Meta+Shift+.`   | `.`†   | `.`‡    | `.`‡   |
| `Alt+Shift+.`    | `˘`    | `˘`     | `˘`    |

### † Chrome reports base character for `.` 

On Chrome `Meta(Cmd)+Shift` and `Ctrl+Shift` reports the base character. 

> Note this is not consistent across all symbols. For example `Ctrl+Shift+[` will report the shifted character `{`.

### ‡ Firefox and Safari difference between `Meta(Cmd)` and `Ctrl`
On Firefox and Safari `Ctrl+Shift` always has an `event.key` of the shifted character. While `Meta(Cmd)+Shift` has the an `event.key` of the base character.

## Digit and letter keys

To show that the shift suppression bites **some digits** but not letters. Here is the `event.key` for different modifiers and platforms. All tested on Chrome. The Windows key was used for Meta. 

| Keys pressed   | macOS | Windows  | ChromeOS |
| -------------- | ----- | -------- | -------- |
| `Shift+1`      | `!`   | `!`      | `!`      |
| `Ctrl+Shift+1` | `1`†  | `!`      | `!`      |
| `Meta+Shift+1` | `1`   | no event | n/a      |
||||
| `Shift+2`      | `@`   | `@`      | `@`      |
| `Ctrl+Shift+2` | `@`†  | `@`      | `@`      |
| `Meta+Shift+2` | `2`   | no event | n/a      |
||||
| `Shift+a`      | `A`   | `A`      | `A`      |
| `Ctrl+Shift+a` | `A`‡  | `A`      | `A`      |
| `Meta+Shift+a` | `a`‡  | no event | n/a      |

### † MacOS digit behavior 
As above with `.`, on MacOS `Ctrl+Shift+1` and `Cmd+Shift+1` reports the base key `1` for the `event.key` whereas other platforms report `!`.

The macOS behavior is **not consistent** across every digit and punctuation key. `Ctrl+Shift+2` reports the **shifted key** `@` for the `event.key`.

### ‡ MacOS letter behavior
`Ctrl+Shift+[letter]` always returns the capital letter. `Cmd+Shift+[letter]` always return the lower case letter.

### Meta key on Windows
The Windows key is reported as the Meta modifier. However `Meta(Windows)+Shift+[digit]` is captured by Windows to do its own shortcuts. `Meta(Windows)+Shift+a` doesn't fire an event or do anything in Windows that I could see.

## MacOS Chrome inconsistency of Ctrl+Shift

Generally digits and symbol keys report their base character on an English keyboard, but there are several exceptions. All letters report their shifted letter, so I'm going to skip those.

| Base Key | Shifted | Ctrl+Shifted |
| -------- | ------- | ------------ |
| `` ` ``  | `~`     | `` ` ``      |
| `1`      | `!`     | `1`          | 
| `2`      | `@`     | `@`†         |
| `3`      | `#`     | `3`          |
| `4`      | `$`     | `4`          |
| `5`      | `%`     | `5`          |
| `6`      | `^`     | `^`†         |
| `7`      | `&`     | `7`          |
| `8`      | `*`     | `8`          |
| `9`      | `(`     | `9`          |
| `0`      | `)`     | `0`          |
| `-`      | `_`     | `_`†         |
| `=`      | `+`     | `=`          |
| `[`      | `{`     | `{`†         |
| `]`      | `}`     | `}`†         |
| `\`      | `\|`    | `\|`†        |
| `;`      | `:`     | `;`          |
| `'`      | `"`     | `'`          |
| `,`      | `<`     | `,`          |
| `.`      | `>`     | `.`          |
| `/`      | `?`     | `/`          |

### † Inconsistent characters

For some reason on a US keyboard `2`, `6`, `-`, `[`, `]`, and `\` report their shifted character. 

## Why macOS reports the unshifted character

**`Cmd` is a macOS-level effect; `Ctrl` is Chrome-specific.** When **Command** is held, macOS's own `event.characters` already resolves to the key's **base** character, so `Cmd+Shift+.` yields `event.key === "."` in **all three browsers** (see the cross-browser table above). **Control** is different: Firefox and Safari still apply Shift (`Ctrl+Shift+.` → `>`), but **Chrome alone returns the base character** (`.`).

Chrome's behavior lives in Chromium's [`DomKeyFromNSEvent`](https://chromium.googlesource.com/chromium/src/+/refs/tags/130.0.6710.0/ui/events/keycodes/keyboard_code_conversion_mac.mm): it derives `event.key` from macOS's `event.characters`, and only when that isn't a usable character does its fallback recompute "with all modifier keys removed except for glyph modifier keys," where glyph modifiers are `Shift | CapsLock | Option` — **Control and Command both excluded**. For a printable punctuation combo like `Ctrl+Shift+.`, `event.characters` is already the base `.`, so the first path returns it and Shift is never re-applied. (This also explains why letters survive: `Ctrl+Shift+a` produces a non-printable control character, so the fallback kicks in *with* Shift and yields `A`. Punctuation gets bitten, letters don't.) The older Cocoa-level rationale — `characters` vs `charactersIgnoringModifiers`, plus the ASCII-hotkey hack — is in the [Chromium OS X keyboard docs](https://www.chromium.org/developers/os-x-keyboard-handling).

This was introduced deliberately in [crbug 586571](https://crbug.com/586571) (2016). The same `Cmd+Shift` behavior is tracked against the other two engines — Firefox ([Bugzilla 1627590](https://bugzilla.mozilla.org/show_bug.cgi?id=1627590)) and WebKit/Safari ([bug 174782](https://bugs.webkit.org/show_bug.cgi?id=174782)) — and as of this writing **both remain open and unfixed**. The Firefox engineer notes the macOS behavior is intentional and that Firefox "cannot change this behavior until Chromium does." So for `Cmd`/`Meta`+`Shift`, returning the base character is still consistent across Chrome, Firefox, and Safari on macOS — which is why Chromium closed its own report ([crbug 40683294](https://issues.chromium.org/issues/40683294)) as WontFix in 2022, deferring the question of what the spec *should* say to [w3c/uievents #169](https://github.com/w3c/uievents/issues/169) (open since 2017).

**The `Ctrl` case is an active Chrome bug.** Because Chrome is the only one of the three browsers that drops Shift for `Ctrl+Shift+`*punctuation* (the `.` table above), it is the outlier there. This is tracked as [crbug 417631300](https://issues.chromium.org/issues/417631300) (P2, active, 2025: `Ctrl+Shift+=` → `=` on macOS Chrome, not reproducible on Windows/Linux); the reporter's note that Firefox and Safari return `+` lines up with the cross-browser table above. This is separate from the `Cmd` case, where all three browsers agree on the base character.

> **Caution — the proposed fix doesn't distinguish `Ctrl` from `Cmd`.** As of patchset 2, the pending patch ([Gerrit 7538394](https://chromium-review.googlesource.com/c/chromium/src/+/7538394)) adds `kNonGlyphModifiers = NSEventModifierFlagControl | NSEventModifierFlagCommand` and, when *either* is held, recomputes `event.key` from the glyph modifiers (Shift/CapsLock/Option) only — returning the **shifted** character and returning before the `event.characters` path. As written, that also changes `Cmd+Shift` (e.g. `Cmd+Shift+.` → `>`, `Cmd+Shift+a` → `A`), which would make Chrome **diverge** from Firefox and Safari for the `Cmd` case (they return the base character) and silently reverse the 2022 WontFix in [crbug 40683294](https://issues.chromium.org/issues/40683294) — the opposite of the CL's stated goal of aligning with Firefox and Safari. Note that [crbug 40683294](https://issues.chromium.org/issues/40683294) itself got the distinction right ("this problem does not exist with Control"); the 2025 fix lost it.

## How to reproduce

Easiest: open <https://keyjs.dev/> and press the combos, reading the `key` field.

Or paste this into any page's console / an HTML file and watch the output:

```html
<pre id="out">press keys…</pre>
<script>
  addEventListener("keydown", (e) => {
    document.getElementById("out").textContent =
      `key=${JSON.stringify(e.key)}  code=${e.code}  ` +
      `shift=${e.shiftKey} ctrl=${e.ctrlKey} alt=${e.altKey} meta=${e.metaKey}`;
    e.preventDefault(); // stop the browser/OS from stealing some combos
  }, { capture: true });
</script>
```

Note: some combos (e.g. `Cmd+Shift+3/4` on macOS, `Ctrl+Shift+T/N` on Windows) are intercepted by the OS/browser and may never reach the page — that's interception, not an `event.key` difference.

## TODO
- comment on https://github.com/w3c/uievents/issues/169: it seems that https://github.com/w3c/uievents/issues/169#issuecomment-343884048 contradicts itself. The native key event's character is not always the shifted value of the character.
- comment on https://issues.chromium.org/issues/417631300: this does not identify the current difference between Ctrl and Cmd on Safari and Firefox. The proposed fix: https://chromium-review.googlesource.com/c/chromium/src/+/7538394 claims to bring Chrome in line with the other browsers but it doesn't fully. And I think it actually shouldn't. The proposed fix is doing what I think should be done. It allows a user to define a Ctrl and Meta(Cmd) hot key based on an actual character value regardless where the character is on the user's keyboard.

## References

- [tinykeys source (v4) — matcher uses `event.key`/`event.code`](https://github.com/jamiebuilds/tinykeys/blob/v4.0.0/src/tinykeys.ts)
- [MDN — `KeyboardEvent.key`](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/key)
- [MDN — Key values for keyboard events](https://developer.mozilla.org/en-US/docs/Web/API/UI_Events/Keyboard_event_key_values)
- [Chromium — OS X keyboard handling](https://www.chromium.org/developers/os-x-keyboard-handling)
- [Chromium — About Mac Hotkeys and Virtual Keycodes](https://chromium.googlesource.com/chromium/src/+/refs/tags/130.0.6710.0/docs/mac/about_hotkeys_and_keycodes.md)
- [Bugzilla 280805 — keypress generates unshifted charcode for Cmd+Shift combos](https://bugzilla.mozilla.org/show_bug.cgi?id=280805)
- [Bugzilla 1627590 — KeyboardEvent.key returns the unshifted value for Cmd+Shift+letter on Mac](https://bugzilla.mozilla.org/show_bug.cgi?id=1627590) — Firefox's tracking bug for the same behavior (filed 2020); **still open/NEW (P5)**. The Firefox engineer calls the macOS behavior intentional and says Firefox can't change it "until Chromium does." Notes that `Ctrl+Shift+L` returns `L` in Firefox while `Cmd+Shift+L` returns `l`.
- [WebKit bug 174782 — `event.key` reports lowercase letter with Shift+Cmd](https://bugs.webkit.org/show_bug.cgi?id=174782) — Safari's version (filed 2017); **still open/NEW**.
- [Chromium issue 40683294 — KeyboardEvent.key holds the wrong value for Command-Shift-modified printable keys](https://issues.chromium.org/issues/40683294) — filed 2020; closed 2022 as WontFix because at the time Firefox and Safari also had the bug and Chromium treated it as consistent cross-browser behavior. Deferred to the spec discussion at w3c/uievents #169.
- [Chromium issue 417631300 — Ctrl+Shift+`=` (`+`) emits wrong value of KeyboardEvent.key on macOS](https://issues.chromium.org/issues/417631300) — filed 2025, P2 active bug with a pending fix. Triage confirmed it reproduces only on macOS Chrome (not Windows/Linux). The reporter's note that Firefox and Safari return `+` for this `Ctrl` combo matches the cross-browser `.` table above (Chrome is the outlier for `Ctrl+Shift`; `Cmd+Shift` is base on all three).
- [Chromium issue 586571 — produce correct DomKey (`event.key`) when Ctrl/Shift/Command is down on Mac](https://crbug.com/586571) ([implementation CL 1706683002](https://codereview.chromium.org/1706683002/)) — the 2016 Chromium CL that *deliberately* introduced this behavior on Mac; claimed to align Mac with Chrome's other platforms, but the tables above show it still diverges for punctuation.
- [Chromium source — `DomKeyFromNSEvent` in `keyboard_code_conversion_mac.mm` (tag 130.0.6710.0)](https://chromium.googlesource.com/chromium/src/+/refs/tags/130.0.6710.0/ui/events/keycodes/keyboard_code_conversion_mac.mm) — the function that computes `event.key` on Mac. It derives the value from macOS's own `event.characters` (already the base glyph for command-type combos); its fallback step removes "all modifier keys … except for glyph modifier keys," where `kGlyphModifiers = NSEventModifierFlagShift | NSEventModifierFlagCapsLock | NSEventModifierFlagOption` — Control and Command are excluded, so they can never re-apply Shift to punctuation.
- [w3c/uievents #169 — Bug in spec? event.key and casing](https://github.com/w3c/uievents/issues/169) — open spec discussion (since 2017) on whether `event.key` should return the shifted character when Cmd/Ctrl is held; Chromium closed issue 40683294 deferring to this thread.
- [w3c/uievents #147 — AltGraph reported as Ctrl+Alt on Windows](https://github.com/w3c/uievents/issues/147)
- [OSXDaily — Option+Shift+/ → ¿ on US Mac](https://osxdaily.com/2022/04/27/type-inverted-question-mark-mac/)
- [github/hotkey #54 — `eventToHotkeyString` inconsistent Shift behaviour on Mac](https://github.com/github/hotkey/issues/54) — a real-world library bug caused by exactly this: on Mac the handler receives the base character plus `shiftKey`, so `?`/`!` get mis-rendered.

---
_Last updated: 2026-05-31_

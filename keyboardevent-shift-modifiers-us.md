---
layout: page
title: "KeyboardEvent.key with Shift + modifiers across platforms (US keyboards)"
description: "How event.key behaves when Shift is combined with Cmd/Ctrl/Alt across macOS, Windows, and ChromeOS — and why character-based shortcut matching breaks for punctuation."
permalink: /keyboardevent-shift-modifiers-us/
---

A reference for how `event.key` behaves when **Shift is combined with a control/command/alt modifier**, and why this makes "bind a shortcut to the character that's actually typed" unreliable across platforms. Relevant to keyboard-shortcut libraries like [tinykeys](https://github.com/jamiebuilds/tinykeys), which can match against `event.key` or `event.code`.

## Scope & assumptions

- **US keyboard layout only.** Other layouts reach these characters differently and will not match this table.
- Platforms tested: **macOS**, **Windows**, **ChromeOS (Chromebook)**.
- Assumed **Chromium-based browser** (Chrome/Edge). Firefox/Safari may differ; note separately if tested.
- **Linux is not a column.** Common desktop Linux (Chromium/X11/Wayland) is expected to track the **ChromeOS** column; verify if it matters to you.
- `event.code` is **not** in the table because it is layout- and modifier-independent: the physical key here is always `Slash` (for `/`), `Digit1`, `KeyA`, etc., on every platform.

## Where key definitions appear

A shortcut definition can show up in three distinct places. They may share a string or differ in format; the guidance below refers to these terms.

- **Canonical form** — the binding's representation in source, in stored config or user preferences, and in the runtime matcher (after parsing). The matching guidance — `event.code` vs `event.key`, per-platform tables — is about what this form contains.
- **User-facing form** — how the binding is presented: the visible label in docs/menus/tooltips, and the `aria-keyshortcuts` attribute exposed to assistive tech. Derived from the Canonical form, with two renderings — platform-styled for visual display, W3C-fixed format for ARIA.
- **Customization input** — how a user (or an author via a UI) supplies a new binding: pressing the combination (recommended — the captured event lands in Canonical form directly), or typing it (most naturally matching the User-facing form, then parsed into Canonical form).

## The data — `/` key (US: `/` unshifted, `?` shifted)

`event.key` for different modifiers and platforms. All tested on Chrome. The Windows key was used for Meta. 

| Keys pressed                   | macOS | Windows | ChromeOS |
| ------------------------------ | ----- | ------- | -------- |
| `/`                            | `/`   | `/`     | `/`      |
| `Shift` + `/`                  | `?`   | `?`     | `?`      |
| `Ctrl` + `Shift` + `/`         | `/`   | `?`     | `?`      |
| `Alt`/`Option` + `Shift` + `/` | `¿`   | `?`     | `?`      |
| `Cmd`/`Meta` + `Shift` + `/`   | `/`   | `?`     | n/a      |

> Note the difference on macOS when a modifier is combined with shift.  Holding **either** `Cmd` or `Ctrl` with `Shift+/` reports the **unshifted** `"/"`, not `"?"`. The `Option+Shift+/` `event.key` actually is the character that is typed while this is different from the other platforms at least we can see which character would be typed.

## Contrast — digit and letter keys

To show that the suppression bites **punctuation** but not letters:

`event.key` for different modifiers and platforms. All tested on Chrome. The Windows key was used for Meta. 

| Keys pressed                   | macOS | Windows  | ChromeOS |
| ------------------------------ | ----- | -------- | -------- |
| `Shift` + `1` (→ `!`)          | `!`   | `!`      | `!`      |
| `Ctrl` + `Shift` + `1`         | `1`   | `!`      | `!`      |
| `Cmd`/`Meta` + `Shift` + `1`   | `1`   | no event | n/a      |
| `Shift` + `a` (→ `A`)          | `A`   | `A`      | `A`      |
| `Ctrl` + `Shift` + `a`         | `A`   | `A`      | `A`      |
| `Cmd`/`Meta` + `Shift` + `a`   | `a`   | no event | n/a      |

> Note on windows the Windows key is reported as the Meta modifier. However `Windows+Shift+[digit]` is captured by Windows to do its own shortcuts. `Windows+Shift+a` doesn't fire an event or do anything in Windows that I could see.

> Note the difference on MacOS with `modifier+shift+1`. It reports `1` for the `event.key` whereas other platforms report `!`. 

**Why letters don't break shortcut matching:** the base (`a`) and shifted (`A`) forms differ only by **case**, and matchers like tinykeys compare case-insensitively — so a binding for `a` matches whether `event.key` comes back `"a"` or `"A"`.

## Discussion

### The original problem

Product requirement: a hotkey should follow **the character that would be typed** (e.g. `?`), not a physical key position. The intuition is that `event.key` already resolves layout + Shift, so matching on `event.key === "?"` should "just work." In isolation it works, but with modifier keys it no longer works on macOS.

### The MacOS problem

**Cmd/Ctrl *suppress* Shift on macOS** When a command-type modifier is held, macOS resolves the event against the key's **base** character, so `Cmd+Shift+/` and `Ctrl+Shift+/` both yield `event.key === "/"`. The precise Cocoa path (`characters` vs `charactersIgnoringModifiers`, plus Chromium's ASCII-hotkey hack for command keys) is described in the Chromium OS X keyboard docs; the observable effect is that the Shift transformation is dropped for punctuation. The same class of bug is recorded against Firefox (Bugzilla 280805).

### Why not just use a mapping `?` <-> `\` on MacOS?

**Layout variance.** On other keyboards layouts the `?` might not require a shift at all. Or it might be the shifted value of a different key (not `/`). 

### What is going on with Alt/Option on MacOS?

**Alt/Option *composes* a different character on macOS.** `Option` is a character-composing modifier (like AltGr). `Option+Shift+/` on a US Mac produces `¿` (inverted question mark) — neither `?` nor `/`. So `event.key` is a *third* character.

### Practical guidance for shortcuts

- **Letters with `$mod`+Shift are safe** (`$mod+Shift+a`) thanks to case-insensitive matching.
- **Modified Shift+punctuation is not portable by character.** You get to pick one:
  - **Match by `event.code`** (e.g. `Slash`) — stable across OS and modifiers, but it is the key's *physical position*, so it ignores the actual typed character. This is hard to describe in user documentation.
  - **Match by character with a per-platform definitions** — honors the typed character, but you maintain platform branches (e.g. `?` on Windows, `/` on Mac).
- **`event.code` is the only primitive that's stable** across all three mechanisms above. The product requirement ("follow the typed character") and "works identically on every platform" are in genuine conflict for modified punctuation shortcuts.

### Across keyboard layouts

The implementation choice has consequences once the user isn't on US QWERTY:

- **`event.code`** is the only stable identifier, but it names a *physical position* — the non-US **User-facing form** has to describe it as "the key where `/` is on a US QWERTY keyboard."
- **By-character matching cannot reach every macOS layout for a symbol hotkey.** Because `Cmd` suppresses the Shift transformation, a user whose layout produces the symbol as a *shifted* character (over a different base) reports the base, not the symbol — so no `Cmd+<symbol>` binding fires for them. Concisely: **on macOS it isn't possible to define a symbol hotkey by character that works for every layout that has that symbol as a base or shifted key.** The "base or shifted" qualifier sidesteps AltGr-only layouts, which are a separate (and similarly difficult) case.

### Guidance

For US Querty keyboards the guidance should be to just use event.code for matching the key. The user documentation can refer to the typed character on a US Querty keyboard. The documentation writer can do the mapping themselves from event.code (plus shift) to typed character.  This means we are not meeting the request of the product owner. 

If that product owner request can't be relaxed, then a lookup table can be used to map typed character to shifted or unshifted code. This lookup table will be US Querty Keyboard specific. So while it will make specification easier for authors, it is locking them into the US Querty Keyboards. This indirection will make supporting other keyboards confusing. The hot key will defined as the US Querty Keyboard typed character but the other keyboard user will need to figure out which physical key that is on their keyboard. For base keys this isn't too bad but for shifted keys you have figure out its unshifted value on a US Query Keyboard and then figure out which physical key that is.

Using this lookup table is preferred to trying to use the `event.key` value directly since that value isn't consistent across platforms.

## ARIA: `aria-keyshortcuts`

This is the second rendering of the **User-facing form** (the visible label is the other), aimed at assistive technology. Its authoring rule is the **opposite** of "follow the typed character" — it documents the keys the user actually presses, not the character that results.

### What it does (and doesn't)

`aria-keyshortcuts` is **declarative metadata only**. Per the spec, "it has no effect on the functionality of the page; the keyboard behavior must be added via JavaScript event handlers." Its job is to tell assistive technology that a shortcut exists so it can be announced to AT users. Everything in the tables and guidance above still applies — you wire up the handler with tinykeys (or whatever); `aria-keyshortcuts` just describes it.

### Syntax

- **Modifier names** (case-insensitive): `Alt`, `Control`, `Shift`, `Meta` (= Cmd on Mac), `AltGraph` (= Option on Mac).
- **Joined with `+`**, modifiers first, exactly **one** non-modifier key last. The literal `+` key is spelled `Plus` (because `+` is the chord delimiter).
- **Non-modifier key**: either a printable character (`A`, `z`, `.`, `$`) **or** a named key from the [UI Events Key Values registry](https://www.w3.org/TR/uievents-key/#named-key-attribute-values) — e.g. `Enter`, `Tab`, `Backspace`, `Delete`, `Escape`, `Home`, `End`, `PageUp`, `PageDown`, `ArrowUp`/`ArrowDown`/`ArrowLeft`/`ArrowRight`, `F1`–`F24`. These are **`KeyboardEvent.key` values, not `event.code` values** — so the `/` key is `"/"`, not `"Slash"`.
- **`Space` is an explicit spec exception.** The real key value is a literal `' '`, which would be parsed as the delimiter between shortcuts — so the spec requires spelling it `Space`.
- **Alphabetic keys are case-insensitive**: `"a"` and `"A"` are equivalent, per spec.
- **Multiple shortcuts**: space-separated, e.g. `aria-keyshortcuts="Alt+Shift+P Control+F"`.
- **HTML escaping**: if a key character would break HTML attribute parsing, use the entity form — e.g. a shortcut on the `'`/`"` key for `"` is `Shift+'`, written as `Shift+&#39;` in the attribute (the spec's own example).
- Applies to all roles; also exposed as `element.ariaKeyShortcuts`.

### The "keys pressed, not the character" rule

The spec is explicit:

> The key combination listed must be the keys the user needs to press, not the outcome of the combined key strokes. For example, on a USA keyboard, if you need the `@` symbol, the key combination is written as `"Shift+2"`, not `"@"` nor `"Shift+@"`.

So for a US-keyboard `?` shortcut, the correct value is **`Shift+/`**, not `?`. This aligns with the `event.code` recommendation in the Guidance above — both describe the keystroke rather than its character output — though `aria-keyshortcuts` uses keycap labels (`Shift+/`) while `event.code` uses physical-key identifiers (`Slash`). The spec also warns authors to "take into account the diversity of available keyboards," i.e., it punts the layout problem back to you.

### Cross-platform

No `$mod`-style abstraction. Either declare both forms or detect platform:

```html
<button aria-keyshortcuts="Control+S Meta+S">Save</button>
```

`AltGraph` is the right token for Mac's Option as a character-composer (e.g. `Option+Shift+/` → `¿`); `Alt` is the plain modifier. Whether AT usefully distinguishes them varies.

### Caveats

- **AT support is variable.** Stable in the spec since ARIA 1.1, but don't assume every screen reader announces it. The spec also recommends surfacing shortcuts visibly in menus/tooltips.
- **Don't shadow AT/OS/browser shortcuts** — doing so can lock AT users out.
- **Disabled elements**: gate the JS handler too; the attribute alone doesn't.

### Concrete example

If your handler fires on `?` typed on a US keyboard:

```html
<button aria-keyshortcuts="Shift+/">Help</button>
```

Not `aria-keyshortcuts="?"`.

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

## References

- [tinykeys source (v4) — matcher uses `event.key`/`event.code`](https://unpkg.com/tinykeys@4.0.0/dist/tinykeys.mjs)
- [MDN — `KeyboardEvent.key`](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/key)
- [MDN — Key values for keyboard events](https://developer.mozilla.org/en-US/docs/Web/API/UI_Events/Keyboard_event_key_values)
- [Chromium — OS X keyboard handling](https://www.chromium.org/developers/os-x-keyboard-handling)
- [Chromium — About Mac Hotkeys and Virtual Keycodes](https://chromium.googlesource.com/chromium/src/+/refs/tags/130.0.6710.0/docs/mac/about_hotkeys_and_keycodes.md)
- [Bugzilla 280805 — keypress generates unshifted charcode for Cmd+Shift combos](https://bugzilla.mozilla.org/show_bug.cgi?id=280805)
- [w3c/uievents #147 — AltGraph reported as Ctrl+Alt on Windows](https://github.com/w3c/uievents/issues/147)
- [OSXDaily — Option+Shift+/ → ¿ on US Mac](https://osxdaily.com/2022/04/27/type-inverted-question-mark-mac/)
- [MDN — `aria-keyshortcuts`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-keyshortcuts)
- [WAI-ARIA 1.2 — `aria-keyshortcuts`](https://www.w3.org/TR/wai-aria-1.2/#aria-keyshortcuts)

---
_Last updated: 2026-05-29_
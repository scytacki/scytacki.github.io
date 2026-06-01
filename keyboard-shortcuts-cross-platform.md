---
layout: page
title: "Working with keyboard shortcuts in browsers: cross-platform pitfalls"
description: "What the per-platform behavior of event.key means when building keyboard shortcuts — why matching on the typed character is unreliable, and what to do instead."
permalink: /keyboard-shortcuts-cross-platform/
---

What the per-platform behavior of `event.key` means when you're building keyboard shortcuts in the browser — why "bind a shortcut to the character that's actually typed" is unreliable across platforms, and what to do instead. Relevant to keyboard-shortcut libraries like [tinykeys](https://github.com/jamiebuilds/tinykeys), which can match against `event.key` or `event.code`.

> This builds on the observed per-platform data in the companion note: [`KeyboardEvent.key` with Shift + modifiers across platforms (US keyboards)](/keyboardevent-shift-modifiers-us/). The tables and reproduction steps live there.

## Where key definitions appear

A shortcut definition can show up in three distinct places. They may share a string or differ in format; the guidance below refers to these terms.

- **Canonical form** — the binding's representation in source, in stored config or user preferences, and in the runtime matcher (after parsing). The matching guidance — `event.code` vs `event.key`, per-platform tables — is about what this form contains.
- **User-facing form** — how the binding is presented: the visible label in docs/menus/tooltips, and the `aria-keyshortcuts` attribute exposed to assistive tech. Derived from the Canonical form, with two renderings — platform-styled for visual display, W3C-fixed format for ARIA.
- **Customization input** — how a user (or an author via a UI) supplies a new binding: pressing the combination (recommended — the captured event lands in Canonical form directly), or typing it (most naturally matching the User-facing form, then parsed into Canonical form).

## Discussion

The [contrast between letters and punctuation](/keyboardevent-shift-modifiers-us/) in the reference tables shows the suppression bites **punctuation** but not letters.

**Why letters don't break shortcut matching:** the base (`a`) and shifted (`A`) forms differ only by **case**, and matchers like tinykeys compare case-insensitively — so a binding for `a` matches whether `event.key` comes back `"a"` or `"A"`.

### The original problem

Product requirement: a hotkey should follow **the character that would be typed** (e.g. `?`), not a physical key position. The intuition is that `event.key` already resolves layout + Shift, so matching on `event.key === "?"` should "just work." In isolation it works, but with modifier keys it no longer works on macOS: holding `Cmd` (or, in Chrome, `Ctrl`) with `Shift+/` reports the unshifted `"/"`, not `"?"`. The reference note explains [why macOS reports the base character](/keyboardevent-shift-modifiers-us/#why-macos-reports-the-unshifted-character) — the Chromium mechanism behind it, and where the browsers stand (the equivalent Firefox and Safari bugs are still open, so `Cmd+Shift` punctuation behaves the same across all three engines on macOS).

### Why not just use a mapping `?` <-> `\` on MacOS?

**Layout variance.** On other keyboards layouts the `?` might not require a shift at all. Or it might be the shifted value of a different key (not `/`). 

### What is going on with Alt/Option on MacOS?

**Option *composes* a different character on macOS.** `Option` is a character-composing modifier (like AltGr). `Option+Shift+/` on a US Mac produces `¿` (inverted question mark) — neither `?` nor `/`. So `event.key` is a *third* character.

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

- **Modifier names** (case-insensitive): `Alt` (= Option on Mac), `Control`, `Shift`, `Meta` (= Cmd on Mac), `AltGraph`. Per the [UI Events Key spec](https://www.w3.org/TR/uievents-key/#keys-modifier), the Mac `Option` key uses the **`Alt`** key value — `AltGraph` is the separate `AltGr` / ISO-Level-3-shift key (common on Windows/Linux non-US layouts), *not* Mac Option.
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

**Use `Alt` for Mac's Option key, not `AltGraph`.** Even though Option *composes* characters the way AltGr does (`Option+Shift+/` → `¿`), the spec maps the Option key to the `Alt` key value; `AltGraph` is reserved for the actual `AltGr` / ISO-Level-3-shift key, which a US Mac doesn't have. And don't expect assistive tech to *localize* the token either: screen readers expose the attribute string essentially verbatim to text-to-speech — there's no layer that turns `Alt` into spoken "Option" for a Mac user. (The NVDA bug tracker shows the value is read as literal text subject to punctuation settings: [#13924](https://github.com/nvaccess/nvda/issues/13924), [#9484](https://github.com/nvaccess/nvda/issues/9484).) On macOS this is largely moot anyway: desktop VoiceOver does not meaningfully announce `aria-keyshortcuts`; the screen readers that do are NVDA and iOS VoiceOver (see [WebAIM: Up and Coming ARIA](https://webaim.org/blog/up-and-coming-aria/)).

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

## References

- [tinykeys source (v4) — matcher uses `event.key`/`event.code`](https://github.com/jamiebuilds/tinykeys/blob/v4.0.0/src/tinykeys.ts)
- [MDN — `aria-keyshortcuts`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-keyshortcuts)
- [WAI-ARIA 1.2 — `aria-keyshortcuts`](https://www.w3.org/TR/wai-aria-1.2/#aria-keyshortcuts)

For the underlying per-platform `event.key` data and the OS-level sources (Chromium OS X handling, Bugzilla 280805, etc.), see the [reference note](/keyboardevent-shift-modifiers-us/#references).

---
_Last updated: 2026-05-30_

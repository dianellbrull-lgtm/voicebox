# Amber Permission-Gate SOP

**Status:** Active — reference procedure, not a pending fix
**Touches:** `app/src/components/AccessibilityGate/`, `app/src/components/InputMonitoringGate/`, `app/src/components/CapturesTab/DictationReadinessChecklist.tsx`, `tauri/src-tauri/src/accessibility.rs`, `tauri/src-tauri/src/input_monitoring.rs`, `app/src/i18n/locales/*/translation.json`
**Last reviewed:** 2026-09-04

## Purpose

Voicebox's dictation pipeline depends on two macOS TCC (Transparency, Consent,
and Control) permissions that have no reliable "denied" signal — the OS just
silently drops behavior instead of erroring:

- **Accessibility** — required for `paste_final_text` to post the synthetic
  ⌘V that lands a transcript in the focused app. Missing it doesn't error;
  `CGEventPost` just no-ops.
- **Input Monitoring** — required for the global chord's `CGEventTap` to see
  key events at all. Missing it doesn't error either; the chord engine just
  never fires.

Because macOS won't tell us "no" out loud, Voicebox surfaces an **amber
warning banner** (`amber-500` border/background, `AlertTriangle` icon) inline
next to whatever control the missing permission breaks. This document is the
standard procedure for how that pattern works, how to extend it to a new
permission, and how to triage user reports about it. "Amber" here always
means *this specific TCC-permission-missing banner* — it is not a general
severity tier.

## Where amber shows up today

There are two permissions, each with a matching gate component:

| Permission | Gate component | Guards |
|---|---|---|
| Accessibility | `AccessibilityGate.tsx` → `AccessibilityNotice` | Auto-paste toggle, Settings → Captures |
| Input Monitoring | `InputMonitoringGate.tsx` → `InputMonitoringNotice` | Global shortcut toggle, Settings → Captures |

Both notices are rendered in `CapturesPage.tsx`, directly under the setting
row they explain, and both render `null` when the permission is already
granted — an amber banner is never shown for a satisfied gate.

There is a **second, separate** surface for the same two permissions:
`DictationReadinessChecklist.tsx`, which shows a green-check/gray-circle row
per readiness gate (models + both TCC permissions) before dictation arms
itself. That checklist deliberately does **not** use amber — a missing gate
there is just "not yet satisfied," not a warning requiring action, so don't
copy amber styling into it. Keep these two surfaces visually distinct:
amber = "you turned this on and it's broken," gray circle = "not satisfied
yet."

## The full pipeline, per permission

### 1. Native check (Rust, `tauri/src-tauri/src/`)

- `accessibility.rs::is_trusted()` calls `AXIsProcessTrusted()` — read-only,
  no prompt side effect. Windows stub returns `true` (no equivalent gate);
  every other non-macOS target returns `false`.
- `input_monitoring.rs::is_trusted()` (named `is_trusted` for symmetry, backed
  by `IOHIDCheckAccess(kIOHIDRequestTypeListenEvent)`) — also read-only.
  `request()` calls `IOHIDRequestAccess`, which queues the system prompt and
  is only invoked once, from `enable_hotkey`, so the OS dialog fires from a
  deterministic user action instead of as a side effect of tap creation.

### 2. Tauri commands exposed to the frontend

`check_accessibility_permission`, `check_input_monitoring_permission`,
`open_accessibility_settings`, `open_input_monitoring_settings` — registered
in `main.rs`'s `invoke_handler`. The `open_*` commands deep-link into the
matching System Settings pane; they do not themselves grant anything.

### 3. React hooks

`useAccessibilityPermission` / `useInputMonitoringPermission` own the
`needsPermission` / `checking` state and re-check on:

- mount (guarded by `platform.metadata.isTauri` — these are no-ops in the
  web build)
- window `focus` (catches the user alt-tabbing back after flipping the
  System Settings toggle)
- accessibility only: the `system:accessibility-missing` Tauri event

### 4. The one live failure signal: `system:accessibility-missing`

Input Monitoring has no live failure signal — a missing grant just means the
chord never fires, which looks identical to "user hasn't pressed the chord
yet," so that gate relies entirely on mount/focus re-checks.

Accessibility gets one extra signal because a paste failure is
distinguishable: `DictateWindow.tsx`'s `paste_final_text` catch block
pattern-matches the error message for `/accessibility/i` and emits
`system:accessibility-missing`, which the main window's `listen()` call
picks up to flip `needsPermission` on immediately — without waiting for a
focus event. If you touch the wording of the Rust-side accessibility error,
keep the string containing "accessibility" (case-insensitive) or this bridge
breaks silently.

### 5. The banner + recheck loop

`AccessibilityNotice` / `InputMonitoringNotice` render the amber block with:

- Title + body copy (via `captures.permissions.<gate>.*` i18n keys)
- "Open Settings" button → the matching `open_*` command
- "I've enabled it" button → `recheck()`, which re-runs the native check
- A `stillMissing` sub-message shown only after an explicit recheck comes
  back still-missing — this is what tells the user macOS usually needs
  Voicebox fully quit and relaunched, not just re-focused, for the grant to
  take effect. Don't show `stillMissing` before the user has actually
  clicked recheck once; it reads as broken software otherwise.

## How to add a new amber-gated permission

1. Add a native `is_trusted()` (+ `request()` if the OS supports prompting)
   in a new `tauri/src-tauri/src/<permission>.rs`, following
   `accessibility.rs`'s doc-comment style — explain *why* the permission is
   needed and what silently breaks without it, not just what the FFI call
   does.
2. Register `check_<permission>_permission` and `open_<permission>_settings`
   commands in `main.rs`.
3. Add a `use<Permission>Permission` hook mirroring
   `useAccessibilityPermission` — mount + focus recheck at minimum; add a
   dedicated Tauri event only if there's a real live failure signal to hang
   it off (see §4 — don't manufacture one).
4. Add a `<Permission>Notice` component using the same amber-500
   border/background/icon classes as the two existing notices, so all TCC
   warnings look like one family.
5. Add `captures.permissions.<permission>.*` keys (`title`, `body`,
   `openSettings`, `recheck`, `rechecking`, `stillMissing`) to **every**
   locale file under `app/src/i18n/locales/`, not just `en`.
6. Add the matching read-only row to `DictationReadinessChecklist.tsx` if
   the permission also blocks dictation arming — reuse `ChecklistRow`, do
   not introduce amber there (see the surface-separation rule above).
7. Render the new `<Permission>Notice` directly under the setting it
   explains, not in a generic "permissions" section — proximity to the
   broken control is the whole point of this pattern.

## Support triage checklist

When a user reports "I granted the permission but Voicebox still says it's
missing":

1. Confirm which gate: Accessibility (paste doesn't happen) or Input
   Monitoring (chord doesn't fire at all).
2. Ask them to fully quit Voicebox (not just close the window) and relaunch.
   This resolves the large majority of these reports — macOS's TCC cache
   frequently doesn't reflect a fresh grant until the process restarts, which
   is exactly what `stillMissing` copy is telling them.
3. If still broken after relaunch, confirm the entry in System
   Settings → Privacy & Security → (Accessibility | Input Monitoring) is
   toggled **on**, not just present in the list — macOS adds the row on
   first prompt but leaves it off until the user flips it.
4. If both of the above check out, have them run the diagnostic below and
   attach the output to the bug report — this distinguishes "grant didn't
   take" from "our check is wrong":

   ```bash
   # Accessibility — 1 means trusted
   osascript -e 'tell application "System Events" to return UI elements enabled'

   # Input Monitoring has no scriptable read; check the TCC db directly
   # (requires Full Disk Access on the terminal app running this)
   sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
     "select client, auth_value from access where service='kTCCServiceListenEvent';"
   ```
5. Known non-bug case: on a fresh install where the user denies the initial
   system prompt outright, macOS will not re-prompt automatically —
   `IOHIDRequestAccess` / `AXIsProcessTrustedWithOptions` only queue a prompt
   once per install. The user must add Voicebox manually via "Open
   Settings," which is exactly what the amber banner's button does; there
   is no code path that can force a second automatic prompt.

## QA checklist before shipping changes near this pattern

- [ ] Toggle the permission off in System Settings, relaunch Voicebox, and
      confirm the amber banner appears next to the correct setting (not just
      in the readiness checklist).
- [ ] Click "Open Settings" and confirm it deep-links to the exact pane
      (Accessibility vs. Input Monitoring — these are easy to swap by
      copy-paste mistake).
- [ ] Grant the permission, click "I've enabled it" **without** relaunching,
      and confirm the banner either clears or shows `stillMissing` — it
      should never silently do nothing.
- [ ] Fully quit and relaunch with the permission granted; confirm the
      banner is gone on first paint (no flash of amber before the mount
      check resolves).
- [ ] Run the same flow on Windows/Linux (or in the web build) and confirm
      no amber banner ever renders — `platform.metadata.isTauri` and the
      Rust stubs returning `true`/`false` are what gate this, so a
      regression here usually means a hook lost its `isTauri` guard.

## Known gotcha: two different "is this macOS" checks

`AccessibilityGate`/`InputMonitoringGate` gate all native calls on
`platform.metadata.isTauri` (correct — these commands don't exist in the web
build). `DictationReadinessChecklist.tsx` instead hides its two TCC rows with
a local `isMacOS` computed by regex-matching `navigator.userAgent`. These are
two different checks answering two different questions ("are we in Tauri at
all" vs. "is this specifically macOS, since Windows/Linux Tauri builds
shouldn't show TCC rows either") — don't consolidate them without confirming
both call sites still get the guard they actually need.

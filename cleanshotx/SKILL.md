---
name: cleanshotx
description: Automates CleanShot X on macOS via its cleanshot:// URL scheme — captures screenshots (area, window, fullscreen, scrolling, previous-area, all-in-one), records the screen, OCRs text from images or screen regions, pins images on top, opens files for annotation, manages capture history, toggles desktop icons, and opens specific settings tabs. Use when the user mentions CleanShot, CleanShot X, cleanshot:// URLs, scripted macOS screenshots or screen recordings, programmatic screen OCR on a Mac, pinning a screenshot, or driving the CleanShot app from a shell, Shortcut, or agent. macOS only. Most commands need CleanShot 3.5.1+; the `action=` parameter and most spatial parameters require 4.7+; `capture-text` requires 3.8.1+ on macOS 10.15+.
license: MIT
compatibility: macOS with CleanShot X installed (https://cleanshot.com). Commands fire by opening cleanshot:// URLs (`open "cleanshot://…"`); requires an Aqua login session and CleanShot to be running or launchable. Over SSH the URL fires only when the same user has an active graphical session. Most commands require 3.5.1+; `action=` and most spatial parameters require 4.7+; `capture-text` requires 3.8.1+ on macOS 10.15+; `all-in-one` 4.2+; `open-history` 4.4+.
metadata:
  app-website: https://cleanshot.com
  api-docs: https://cleanshot.com/docs-api
  url-scheme: cleanshot://
---

# Driving CleanShot X via the cleanshot:// URL scheme

CleanShot X is a macOS screenshot and screen-recording app from MagicLasso. It exposes a URL scheme (`cleanshot://command?param=value&…`) that triggers any of its capture, recording, OCR, annotation, pin, history, or settings actions. Opening such a URL launches the app (if needed) and runs the action. That URL scheme is the **only** supported automation surface — there is no public CLI, AppleScript dictionary, or HTTP API. macOS Shortcuts.app actions internally invoke the same scheme.

The full per-command reference with every parameter and version constraint lives in [references/url-scheme.md](references/url-scheme.md). Read it before automating any command this `SKILL.md` does not show in full.

## Gotchas — read these before automating

- **Requires an Aqua login session.** `open "cleanshot://…"` goes through LaunchServices and the app's UI. It needs an active logged-in graphical session for the same user — over SSH it fires only when that user is also logged in locally. It will not run from a launchd `Background` agent or before login. Use a user-context launchd agent (`LimitLoadToSessionType: Aqua`), Screen Sharing, or run it as a step inside a logged-in session.
- **CleanShot X must be installed and able to launch.** If the app is not installed, opening the URL silently does nothing or prompts the user to choose an app. Verify with `mdfind "kMDItemCFBundleIdentifier == 'pl.maketheweb.cleanshotx'"`, or check the common install paths: `/Applications/CleanShot X.app` (direct/App Store) and `~/Applications/Setapp/CleanShot X.app` (Setapp).
- **No return value, no completion signal.** The URL fires asynchronously; the calling shell gets exit code 0 as soon as `open` hands off. There is no built-in way to wait for "capture finished" or to recover the resulting file path. To know what happened: inspect the configured save folder (CleanShot ▸ Settings ▸ Screenshots ▸ "Save to") or watch the clipboard.
- **URL parameters must be URL-encoded.** Spaces in a `filepath` become `%20`. Always quote the URL when passing it to `open`, and percent-encode user-supplied paths. See the helper at the bottom of this file.
- **Version-gated parameters may not work on older builds.** Parameters added in 4.7 (`action=…`, `x/y/width/height/display` on most commands, `start=true`, `autoscroll=true`, `tab=…`) are not supported on earlier versions and are typically ignored — the command still runs without the parameter. Don't rely on a version-gated parameter without first checking `defaults read "/Applications/CleanShot X.app/Contents/Info" CFBundleShortVersionString`.
- **`action` is a single value, not a chain.** `action=annotate` alone is valid; `action=copy,upload` is **not**. Pick one of `copy | save | annotate | upload | pin`. To compose actions (e.g. annotate then upload), drive subsequent steps from a follow-up URL or from the annotator UI.
- **`filepath` parameter rules.** When you supply a `filepath` to any command that takes one (`pin`, `open-annotate`, `capture-text`, `add-quick-access-overlay`), it must be an **absolute path** — not `~`-relative. Expand `~` in the shell before passing. The parameter is **optional** for `pin` / `open-annotate` / `capture-text` (omitting it falls back to an interactive picker or area selection — see [references/url-scheme.md](references/url-scheme.md)) and **required** for `add-quick-access-overlay`. Accepted formats: PNG and JPEG for screenshot commands; `add-quick-access-overlay` additionally accepts MP4.
- **`capture-window` is always interactive — wrong tool for "screenshot the X window" automation.** It enters a hover-and-click selection mode; there is no parameter to target a window by name, PID, or window ID. For unattended capture of a known window, use `capture-area` with explicit `x`, `y`, `width`, `height` (4.7+) computed from the window's frame — see [Capture a specific named window unattended](#capture-a-specific-named-window-unattended) below. Do **not** drop down to `screencapture -l` just because `capture-window` does not fit; CleanShot's URL scheme can do this headlessly.
- **Apple-backend / sandbox surprises.** Capturing a window or area may require Screen Recording permission for the app. Recording the screen also requires Microphone permission if audio is enabled. The first run of any command may surface a TCC prompt that needs a user click — agents cannot auto-accept TCC.
- **`scrolling-capture` only proceeds with `start=true`** on 4.7+; without it the user has to click "Start" in the overlay. Use `start=true&autoscroll=true` for a fully unattended capture (4.7+).
- **`record-screen` does not stop itself.** There is no `stop-recording` URL. Stop recording from the menu-bar item, the global shortcut (default ⌘⇧⌥3 stop, or click the floating control), or by sending a `cleanshot://record-screen` *toggle* — but the toggle behavior depends on the user's settings and is not guaranteed. Treat recording as user-supervised.
- **`display=` is a 1-based index** of the display ordering CleanShot sees. Display 1 is typically the main display; multi-display indices may not match `system_profiler SPDisplaysDataType` ordering. Test on the target machine.
- **No undo for `delete`-style actions inside CleanShot.** The URL scheme has no destructive commands directly, but `restore-recently-closed` and `open-history` both touch the history database — back it up before bulk operations.

## Quick start: invoking the URL scheme

The single primitive is `open` with a `cleanshot://` URL. From a shell:

```bash
# Launch the area-selection capture, then open the result in the annotator.
open "cleanshot://capture-area?action=annotate"

# Repeat the previous-area capture and copy the result to the clipboard.
open "cleanshot://capture-previous-area?action=copy"

# OCR text out of an existing image and copy it to the clipboard.
open "cleanshot://capture-text?filepath=/Users/me/Desktop/screenshot.png&linebreaks=true"

# Pin an image on top of all windows.
open "cleanshot://pin?filepath=/Users/me/Desktop/diagram.png"

# Open the Recording tab in CleanShot's settings.
open "cleanshot://open-settings?tab=recording"
```

From AppleScript / `osascript`:

```bash
osascript -e 'open location "cleanshot://capture-fullscreen?action=upload"'
```

From Python (when you need URL-encoding done for you):

```python
import subprocess, urllib.parse, pathlib
path = pathlib.Path("~/Desktop/My Screenshot.png").expanduser().resolve()
url = "cleanshot://pin?filepath=" + urllib.parse.quote(str(path), safe="/")
subprocess.run(["open", url], check=True)
```

## Command catalog (summary)

The 19 commands fall into seven groups. For full parameters, defaults, and per-parameter version requirements, see [references/url-scheme.md](references/url-scheme.md).

| Group | Commands |
|---|---|
| Screenshots | `capture-area`, `capture-previous-area`, `capture-fullscreen`, `capture-window`, `self-timer`, `scrolling-capture`, `pin` |
| Recording | `record-screen` |
| OCR | `capture-text` |
| Annotation | `open-annotate`, `open-from-clipboard` |
| All-In-One | `all-in-one` |
| Desktop icons | `toggle-desktop-icons`, `hide-desktop-icons`, `show-desktop-icons` |
| History / overlays / settings | `add-quick-access-overlay`, `open-history`, `restore-recently-closed`, `open-settings` |

Common parameter shapes:

- `action=copy | save | annotate | upload | pin` — what to do with the result. Requires CleanShot 4.7+. Default is whatever the user has configured in CleanShot ▸ Settings ▸ Screenshots ▸ "After capture".
- `x`, `y`, `width`, `height`, `display` — capture region. Requires 4.7+ on most commands. Coordinates use the macOS native coordinate system: **origin (0,0) is the lower-left corner of the screen**, with `y` increasing upward (per the CleanShot docs). Units are points, not Retina pixels. `display` is a 1-based integer; if omitted it defaults to the display under the cursor.
- `filepath` — absolute path to a PNG or JPEG (MP4 also for `add-quick-access-overlay`). Must be URL-encoded.

## Common workflows

### One-shot: capture an area and upload to the cloud

```bash
open "cleanshot://capture-area?action=upload"
```

The user drags out a region; CleanShot uploads to the configured destination (CleanShot Cloud, S3, Dropbox, etc.) and copies the resulting URL to the clipboard. The URL is *not* returned to the caller — read it from the clipboard:

```bash
pbpaste   # → https://cln.sh/abcd1234
```

### Repeat the previous capture region and save to disk

Useful for taking a sequence of "same region" shots while a UI updates:

```bash
open "cleanshot://capture-previous-area?action=save"
```

### OCR text out of an image file

```bash
open "cleanshot://capture-text?filepath=$(python3 -c 'import urllib.parse,sys;print(urllib.parse.quote(sys.argv[1]))' "$HOME/Downloads/menu.jpg")&linebreaks=false"
sleep 1
pbpaste   # OCR'd text
```

`linebreaks=false` flattens the text into a single paragraph; `true` preserves line breaks. CleanShot's standard OCR flow writes the recognized text to the clipboard — read it with `pbpaste` (the URL itself returns nothing).

### Capture a specific named window unattended

The right tool for "screenshot Ghostty's window" with no clicks is **not** `capture-window` (that command always opens an interactive picker). It is `capture-area` with explicit `x/y/width/height` (4.7+) computed from the window's frame, plus `action=save`. The only twist is the coordinate flip: macOS UI APIs (System Events, AppKit, Accessibility) report top-left-origin bounds, but CleanShot's URL scheme uses the lower-left origin (`cs_y = display_height - top_y - height`).

```bash
APP="Ghostty"

# 1. Pull front-window position/size via System Events (top-left origin, points).
read -r WX WY WW WH <<<"$(osascript <<EOF
tell application "System Events" to tell process "$APP"
  set p to position of window 1
  set s to size of window 1
  return (item 1 of p as text) & " " & (item 2 of p as text) & " " & (item 1 of s as text) & " " & (item 2 of s as text)
end tell
EOF
)"

# 2. Read the main display height in points (Quartz, not Retina pixels).
SH=$(osascript -e 'tell application "Finder" to get bounds of window of desktop' \
  | awk -F', ' '{print $4}')

# 3. Flip the y origin: CleanShot measures from the bottom of the screen.
CSY=$(( SH - WY - WH ))

# 4. Fire the headless region capture and save to disk.
open "cleanshot://capture-area?x=${WX}&y=${CSY}&width=${WW}&height=${WH}&action=save"
```

Add `&display=N` (1-based) if the window lives on a non-primary display. The capture itself is unattended; CleanShot may still flash a save-confirmation overlay unless the user has set "After capture: Save" in their settings. To find the resulting file path, see [Locate the file CleanShot just saved](#locate-the-file-cleanshot-just-saved) below.

If you need the window's bounds programmatically without AppleScript, `Quartz.CGWindowListCopyWindowInfo` (PyObjC) returns each window's `kCGWindowBounds`; convert the same way.

### Unattended scrolling capture of a long page (4.7+)

```bash
open "cleanshot://scrolling-capture?start=true&autoscroll=true"
```

Without `autoscroll=true`, the user has to scroll manually; without `start=true`, they have to click Start in the overlay.

### Pin an image as a floating reference

```bash
ABS_PATH=$(python3 -c 'import os,sys;print(os.path.abspath(os.path.expanduser(sys.argv[1])))' "~/Desktop/spec.png")
ENC_PATH=$(python3 -c 'import urllib.parse,sys;print(urllib.parse.quote(sys.argv[1]))' "$ABS_PATH")
open "cleanshot://pin?filepath=$ENC_PATH"
```

The image floats above all windows until the user closes the pin. The URL scheme has no documented unpin command — close the pin manually or via UI scripting.

### Locate the file CleanShot just saved

The `open "cleanshot://…"` call returns immediately and prints nothing — there is no synchronous way to recover the saved file path. Three workable patterns:

1. **Poll the configured save folder.** Read the user's "Save to" path and pick the newest file written after a marker:

   ```bash
   SAVE_DIR=$(defaults read pl.maketheweb.cleanshotx CaptureFolder 2>/dev/null \
     || echo "$HOME/Desktop")
   touch /tmp/.cs.marker
   open "cleanshot://capture-area?x=0&y=0&width=800&height=600&action=save"
   # Wait briefly for CleanShot to write the file, then grab the newest one.
   for _ in 1 2 3 4 5; do
     NEW=$(find "$SAVE_DIR" -type f -newer /tmp/.cs.marker \
       \( -name '*.png' -o -name '*.jpg' \) -print -quit)
     [ -n "$NEW" ] && { echo "$NEW"; break; }
     sleep 0.5
   done
   ```

   `find … -newer` works because CleanShot writes a brand-new file per capture.

2. **Use `action=copy` instead of `save`.** The image goes on the clipboard; pull bytes with `pbpaste` (or the AppKit `NSPasteboard`) and write them yourself.

3. **Use `action=upload`.** CleanShot copies the upload URL (not the image) to the clipboard — `pbpaste` returns it, e.g. `https://cln.sh/abcd1234`.

Do **not** assume `open` prints the file path: it never does, regardless of `action=`.

### Detect installed version before using 4.7-only parameters

```bash
VERSION=$(defaults read "/Applications/CleanShot X.app/Contents/Info" CFBundleShortVersionString 2>/dev/null || echo "0")
# Compare with sort -V; bail out cleanly if too old.
if [ "$(printf '%s\n4.7\n' "$VERSION" | sort -V | head -n1)" != "4.7" ]; then
  echo "CleanShot $VERSION is too old for action= / spatial parameters" >&2
  exit 1
fi
```

### URL-encode a path safely (helper)

CleanShot accepts only percent-encoded URLs. The most reliable encoder for an arbitrary path:

```bash
encode_path() {
  python3 -c 'import sys,urllib.parse,os;print(urllib.parse.quote(os.path.abspath(os.path.expanduser(sys.argv[1])),safe="/"))' "$1"
}
open "cleanshot://open-annotate?filepath=$(encode_path "~/Pictures/My Shot.png")"
```

Bash-only fallback (no Python): `printf '%s' "$path" | jq -sRr @uri` if `jq` is available.

## Verifying CleanShot is present and runnable

Before firing any URL, agents should confirm:

```bash
# 1. App installed? (covers /Applications and Setapp)
if [ ! -d "/Applications/CleanShot X.app" ] && [ ! -d "$HOME/Applications/Setapp/CleanShot X.app" ]; then
  # Fallback: query LaunchServices by bundle ID (also handles non-standard paths)
  if [ -z "$(mdfind "kMDItemCFBundleIdentifier == 'pl.maketheweb.cleanshotx'")" ]; then
    echo "CleanShot X not installed" >&2; exit 1
  fi
fi

# 2. URL scheme registered? (LaunchServices)
/System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/LaunchServices.framework/Versions/A/Support/lsregister -dump 2>/dev/null \
  | grep -A1 "scheme:" | grep -q "cleanshot" \
  && echo "URL scheme registered"

# 3. Logged-in graphical session? (must be Aqua)
launchctl managername | grep -qi Aqua || echo "Warning: not an Aqua session — URL scheme may not fire"
```

## When the URL scheme isn't enough

Things the URL scheme **cannot** do — escalate or document the limitation rather than inventing a command:

- Programmatically stop a recording (no `stop-recording` URL).
- Compose multiple post-capture actions in one call (`action=copy,upload`).
- Read the resulting file path or upload URL from the command — must read the clipboard or scan the configured save folder.
- Drive Quick Access overlay placement, size, or z-order beyond "added".
- Change settings programmatically (only *open* the settings tab).
- Capture without a user-visible UI (every command surfaces some on-screen affordance).
- Unpin a pinned image.

For workflows that need richer integration than the URL scheme exposes, fall back to **macOS Shortcuts.app** (CleanShot ships several Shortcut actions that wrap these URLs). Drop down to OS-level alternatives (`screencapture`, `xcrun simctl io … screenshot`, native AVFoundation recorders) only when CleanShot is genuinely unavailable. In particular, do **not** reach for `screencapture -l <windowID>` to dodge `capture-window`'s click — `cleanshot://capture-area` with explicit `x/y/w/h` (see [Capture a specific named window unattended](#capture-a-specific-named-window-unattended)) covers that case headlessly.

## Where to look next

- [references/url-scheme.md](references/url-scheme.md) — every command, every parameter, version requirements, examples.
- Vendor docs: <https://cleanshot.com/docs-api> (authoritative; check for new commands when this skill seems incomplete).
- App version: `defaults read "/Applications/CleanShot X.app/Contents/Info" CFBundleShortVersionString`.
- App support directory (per-user CleanShot state): `~/Library/Application Support/pl.maketheweb.cleanshotx/`.
- Default save folder for captured files: shown in CleanShot ▸ Settings ▸ Screenshots ▸ "Save to".

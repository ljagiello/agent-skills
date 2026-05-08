# CleanShot X cleanshot:// URL scheme — full command reference

Every public command exposed by CleanShot X, with parameters, version requirements, and a worked example. Source: <https://cleanshot.com/docs-api>. Verify against that page if a command appears to behave differently — Magic Lasso adds new parameters between minor versions.

## Contents

1. [`capture-area`](#1-capture-area) — area-selection screenshot
2. [`capture-previous-area`](#2-capture-previous-area) — repeat the last area capture
3. [`capture-fullscreen`](#3-capture-fullscreen) — full screen, no UI
4. [`capture-window`](#4-capture-window) — pick a window
5. [`self-timer`](#5-self-timer) — area capture with a countdown
6. [`scrolling-capture`](#6-scrolling-capture) — long-page / scrollable capture
7. [`pin`](#7-pin) — pin an image as a floating window
8. [`record-screen`](#8-record-screen) — start a screen recording
9. [`capture-text`](#9-capture-text) — OCR an image or screen region
10. [`open-annotate`](#10-open-annotate) — open an image in the annotator
11. [`open-from-clipboard`](#11-open-from-clipboard) — annotate the clipboard image
12. [`all-in-one`](#12-all-in-one) — unified capture/recording overlay
13. [`toggle-desktop-icons`](#13-toggle-desktop-icons)
14. [`hide-desktop-icons`](#14-hide-desktop-icons)
15. [`show-desktop-icons`](#15-show-desktop-icons)
16. [`add-quick-access-overlay`](#16-add-quick-access-overlay) — add a file to the Quick Access overlay
17. [`open-history`](#17-open-history)
18. [`restore-recently-closed`](#18-restore-recently-closed)
19. [`open-settings`](#19-open-settings) — open a specific settings tab

Plus: [Quick parameter index](#quick-parameter-index) and [`action=` value behavior](#behavior-of-action-values) at the end.

## Conventions

- All commands use the form `cleanshot://<command>?param1=value1&param2=value2`.
- Fire with `open "cleanshot://…"` from a logged-in macOS Aqua session.
- All parameter values must be URL-encoded (spaces → `%20`, `&` → `%26`, etc.).
- Values shown as `true | false` accept the literal lowercase strings.
- `display=` is a 1-based integer index (`1, 2, …`) of the displays as CleanShot sees them. If omitted, CleanShot defaults to the display under the cursor.
- **Coordinate system:** origin `(0,0)` is the **lower-left** corner of the screen, with `y` increasing upward (per the CleanShot docs). Spatial parameters (`x`, `y`, `width`, `height`) are integers in points, not Retina pixels.
- Version cells refer to the **CleanShot X version** the command (or that specific parameter) first appeared in. macOS minimums are called out where they apply.
- The `action=` parameter, when present, accepts exactly one of: `copy`, `save`, `annotate`, `upload`, `pin`. It overrides the user's "After capture" default for that one invocation.

---

## 1. `capture-area`

Open the standard area-selection capture overlay.

| Parameter | Type | Required | Min version | Notes |
|---|---|---|---|---|
| `x` | integer | no | 4.7+ | x of the pre-selected region (points; lower-left origin). |
| `y` | integer | no | 4.7+ | y of the pre-selected region (points; measured from the bottom of the screen). |
| `width` | integer | no | 4.7+ | Width of pre-selected region. |
| `height` | integer | no | 4.7+ | Height of pre-selected region. |
| `display` | integer | no | 4.7+ | 1-based display index. |
| `action` | enum | no | 4.7+ | `copy \| save \| annotate \| upload \| pin`. |

Min CleanShot version: **3.5.1+**.

Examples:

```text
cleanshot://capture-area
cleanshot://capture-area?action=annotate
cleanshot://capture-area?x=100&y=200&width=640&height=480&display=1&action=upload
```

If all four spatial parameters are supplied, CleanShot still shows the selection rectangle pre-positioned but allows the user to adjust before confirming — there is no documented "fire immediately" mode for a non-fullscreen area.

---

## 2. `capture-previous-area`

Repeat the most recent area capture using the previous coordinates.

| Parameter | Type | Required | Min version | Notes |
|---|---|---|---|---|
| `action` | enum | no | 4.7+ | `copy \| save \| annotate \| upload \| pin`. |

Min CleanShot version: **3.5.1+**.

Example:

```text
cleanshot://capture-previous-area?action=copy
```

If there is no "previous area" yet (first run after launch), CleanShot falls back to a normal area capture.

---

## 3. `capture-fullscreen`

Capture the entire screen immediately — no UI, no selection.

| Parameter | Type | Required | Min version | Notes |
|---|---|---|---|---|
| `action` | enum | no | 4.7+ | `copy \| save \| annotate \| upload \| pin`. |

Min CleanShot version: **3.5.1+**.

Example:

```text
cleanshot://capture-fullscreen?action=save
```

There is no per-display restriction — fullscreen captures every connected display unless the user has disabled "Capture all displays" in settings.

---

## 4. `capture-window`

Enter window-selection capture mode (hover and click a window to capture it).

| Parameter | Type | Required | Min version | Notes |
|---|---|---|---|---|
| `action` | enum | no | 4.7+ | `copy \| save \| annotate \| upload \| pin`. |

Min CleanShot version: **3.5.1+**.

Example:

```text
cleanshot://capture-window?action=annotate
```

Requires Screen Recording permission for CleanShot to read window contents on macOS 10.15+.

---

## 5. `self-timer`

Open the area capture overlay with the self-timer enabled. The user picks the area; CleanShot counts down before snapping.

| Parameter | Type | Required | Min version | Notes |
|---|---|---|---|---|
| `action` | enum | no | 4.7+ | `copy \| save \| annotate \| upload \| pin`. |

Min CleanShot version: **3.5.1+**.

Example:

```text
cleanshot://self-timer?action=upload
```

The countdown duration is whatever the user has configured in CleanShot settings; the URL scheme cannot override it.

---

## 6. `scrolling-capture`

Open the scrolling-capture overlay for capturing long pages, scrollable lists, etc.

| Parameter | Type | Required | Min version | Notes |
|---|---|---|---|---|
| `x` | integer | no | 4.7+ | x of the pre-selected region (lower-left origin). |
| `y` | integer | no | 4.7+ | y of the pre-selected region (from screen bottom). |
| `width` | integer | no | 4.7+ | Region width. |
| `height` | integer | no | 4.7+ | Region height. |
| `display` | integer | no | 4.7+ | 1-based display index. |
| `start` | bool | no | 4.7+ | `true` to begin capturing immediately without the user clicking Start. |
| `autoscroll` | bool | no | 4.7+ | `true` to make CleanShot drive the scroll itself instead of the user. |

Min CleanShot version: **3.5.1+**. Spatial / `start` / `autoscroll` parameters require **4.7+**.

Examples:

```text
cleanshot://scrolling-capture
cleanshot://scrolling-capture?start=true
cleanshot://scrolling-capture?start=true&autoscroll=true
cleanshot://scrolling-capture?x=0&y=0&width=1280&height=800&display=1&start=true&autoscroll=true
```

`autoscroll=true` only works for windows whose scroll area CleanShot can drive — most native macOS apps and Chromium/WebKit browsers. Some Electron apps and custom views require manual scroll.

---

## 7. `pin`

Pin an image as a floating window above all other windows.

| Parameter | Type | Required | Min version | Notes |
|---|---|---|---|---|
| `filepath` | absolute path | no | 3.5.1+ | PNG or JPEG. If omitted, CleanShot prompts the user to pick a file (per the docs). |

Min CleanShot version: **3.5.1+**.

Examples:

```text
cleanshot://pin
cleanshot://pin?filepath=/Users/john/Desktop/my%20screenshot.png
```

There is **no** URL to programmatically unpin; the user must close the pin window manually.

---

## 8. `record-screen`

Start a screen recording. Whether the recording begins immediately or shows a region overlay depends on the user's CleanShot settings.

| Parameter | Type | Required | Min version | Notes |
|---|---|---|---|---|
| `x` | integer | no | 4.7+ | x of the recording region (lower-left origin). |
| `y` | integer | no | 4.7+ | y of the recording region (from screen bottom). |
| `width` | integer | no | 4.7+ | Region width. |
| `height` | integer | no | 4.7+ | Region height. |
| `display` | integer | no | 4.7+ | 1-based display index. Capture entire display when no x/y/width/height. |

Min CleanShot version: **3.5.1+**. Spatial parameters require **4.7+**.

Examples:

```text
cleanshot://record-screen
cleanshot://record-screen?display=1
cleanshot://record-screen?x=0&y=0&width=1920&height=1080&display=1
```

There is **no** stop-recording URL. Stop via the menu-bar item, the floating recording control, or the user's configured shortcut. Audio recording (microphone or system audio) follows the user's settings — the URL cannot override them.

---

## 9. `capture-text`

Run OCR on either an existing image or a region of the screen. CleanShot's standard OCR flow places the recognized text on the clipboard.

| Parameter | Type | Required | Min version | Notes |
|---|---|---|---|---|
| `filepath` | absolute path | no | 3.8.1+ | PNG or JPEG. If omitted, CleanShot opens an interactive selection (area overlay) for OCR — the docs do not enumerate the exact UI but the spatial parameters below imply a region selection. |
| `x` | integer | no | 4.7+ | x of the region to OCR (only when `filepath` omitted; lower-left origin). |
| `y` | integer | no | 4.7+ | y of the region to OCR (from screen bottom). |
| `width` | integer | no | 4.7+ | Region width. |
| `height` | integer | no | 4.7+ | Region height. |
| `display` | integer | no | 4.7+ | 1-based display index. |
| `linebreaks` | bool | no | 3.8.1+ | `true` keeps line breaks in the recognized text; `false` flattens to a single paragraph. |

Min CleanShot version: **3.8.1+**. macOS minimum: **10.15+** (uses Apple's Vision framework).

Examples:

```text
cleanshot://capture-text
cleanshot://capture-text?linebreaks=true
cleanshot://capture-text?filepath=/Users/john/Desktop/screenshot.png&linebreaks=false
```

In CleanShot's standard OCR flow the recognized text lands on the clipboard; there is no parameter to write it to a file. Pipe `pbpaste` from the calling shell after the URL fires.

---

## 10. `open-annotate`

Open an image in CleanShot's annotator.

| Parameter | Type | Required | Min version | Notes |
|---|---|---|---|---|
| `filepath` | absolute path | no | 3.8.1+ | PNG or JPEG. If omitted, CleanShot prompts the user to pick a file (per the docs). |

Min CleanShot version: **3.8.1+**.

Example:

```text
cleanshot://open-annotate?filepath=/Users/john/Desktop/image.png
```

---

## 11. `open-from-clipboard`

Open whatever image is on the clipboard in the annotator.

No parameters. Min CleanShot version: **3.5.1+**.

Example:

```text
cleanshot://open-from-clipboard
```

If the clipboard does not contain an image, CleanShot shows a "no image on clipboard" banner and exits.

---

## 12. `all-in-one`

Open the All-In-One mode (a unified capture/recording overlay added in 4.x).

| Parameter | Type | Required | Min version | Notes |
|---|---|---|---|---|
| `x` | integer | no | 4.7+ | x of the pre-selected region (lower-left origin). |
| `y` | integer | no | 4.7+ | y of the pre-selected region (from screen bottom). |
| `width` | integer | no | 4.7+ | Region width. |
| `height` | integer | no | 4.7+ | Region height. |
| `display` | integer | no | 4.7+ | 1-based display index. |

Min CleanShot version: **4.2+**. Spatial parameters require **4.7+**.

Examples:

```text
cleanshot://all-in-one
cleanshot://all-in-one?x=100&y=120
cleanshot://all-in-one?x=0&y=0&width=1280&height=800&display=1
```

---

## 13. `toggle-desktop-icons`

Toggle the visibility of macOS desktop icons.

No parameters. Min CleanShot version: **3.5.1+**.

Example:

```text
cleanshot://toggle-desktop-icons
```

---

## 14. `hide-desktop-icons`

Hide macOS desktop icons (idempotent — safe to call when already hidden).

No parameters. Min CleanShot version: **3.8.1+**.

Example:

```text
cleanshot://hide-desktop-icons
```

---

## 15. `show-desktop-icons`

Show macOS desktop icons (idempotent).

No parameters. Min CleanShot version: **3.8.1+**.

Example:

```text
cleanshot://show-desktop-icons
```

---

## 16. `add-quick-access-overlay`

Add a file to the Quick Access Overlay (a per-screen palette of recent media that CleanShot can keep visible).

| Parameter | Type | Required | Min version | Notes |
|---|---|---|---|---|
| `filepath` | absolute path | **yes** | 3.8.1+ | PNG, JPEG, or MP4. |

Min CleanShot version: **3.8.1+**.

Example:

```text
cleanshot://add-quick-access-overlay?filepath=/Users/john/Desktop/screenshot.png
```

Without `filepath`, the URL is a no-op.

---

## 17. `open-history`

Open CleanShot's capture history window.

No parameters. Min CleanShot version: **4.4+**.

Example:

```text
cleanshot://open-history
```

---

## 18. `restore-recently-closed`

Re-open the most recently closed CleanShot capture or pin (the docs do not enumerate the full set of items this command tracks).

No parameters. Min CleanShot version: **3.5.1+**.

Example:

```text
cleanshot://restore-recently-closed
```

---

## 19. `open-settings`

Open the CleanShot Settings window, optionally focused on a specific tab.

| Parameter | Type | Required | Min version | Notes |
|---|---|---|---|---|
| `tab` | enum | no | 4.7+ | One of: `general`, `wallpaper`, `shortcuts`, `quickaccess`, `recording`, `screenshots`, `annotate`, `cloud`, `advanced`, `about`. |

Min CleanShot version: **4.7+**.

Examples:

```text
cleanshot://open-settings
cleanshot://open-settings?tab=recording
cleanshot://open-settings?tab=cloud
```

Unknown `tab` values fall back to the General tab on most builds.

---

## Quick parameter index

| Parameter | Used by |
|---|---|
| `action` | `capture-area`, `capture-previous-area`, `capture-fullscreen`, `capture-window`, `self-timer` |
| `x`, `y`, `width`, `height`, `display` | `capture-area`, `scrolling-capture`, `record-screen`, `capture-text`, `all-in-one` |
| `start`, `autoscroll` | `scrolling-capture` |
| `linebreaks` | `capture-text` |
| `filepath` | `pin`, `capture-text`, `open-annotate`, `add-quick-access-overlay` |
| `tab` | `open-settings` |

## Behavior of `action=` values

| Value | What CleanShot does after the capture |
|---|---|
| `copy` | Copies the image to the clipboard. No save dialog. |
| `save` | Writes the image to the configured save folder using the user's filename template. |
| `annotate` | Opens the result in the CleanShot annotator. |
| `upload` | Uploads to the configured destination (CleanShot Cloud / S3 / Dropbox / etc.) and copies the resulting URL to the clipboard. |
| `pin` | Pins the image as a floating window. |

`action=` is single-valued. To compose actions, drive a follow-up URL after the first one resolves (e.g. capture with `action=save`, then `cleanshot://open-annotate?filepath=…` against the saved file).

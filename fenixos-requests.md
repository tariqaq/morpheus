# fenixOS feature requests from Morpheus

For the Claude Code session that works on `tariqaq/fenixOS`. Read this, check each request against the current code and docs (`docs/packages.md`, `proto/spec.md`, `docs/sidecars.md`, `docs/registry.md`, `DESIGN.md`), and say for each one whether fenixOS can support it, how, and in which release.

## Context

Morpheus (`tariqaq/morpheus`, id `io.github.tariqaq.morpheus` planned) is a fenixOS app. Its board screen **morphs to match what the user is doing on the PC**:

- **Media / video:** album art, title and artist, live seek bar, play/pause, skip, volume.
- **Game:** game name, session time, stats.
- **Call:** mute, deafen, call timer.
- **Glance:** anything else, described in a few short AI-written lines.

The PC half (Rust, `fenix-sdk`) works out the context. It uses Windows APIs (SMTC for media, Core Audio, the foreground window) plus DeepSeek reading screenshots. It then drives the board with `set`.

Checked against fenixOS `main` (latest tag v0.1.19) on 2026-11-01. Morpheus was made from the app template for v0.1.13.

## What blocks it today

| Need | Today | Where |
|---|---|---|
| Switch between whole screens (media / game / call / glance) at runtime | `hidden` is a compile-time prop (`hidden: 0/1`, prop 22). Nothing can show or hide an element at runtime | `docs/packages.md` (props, binary prop table) |
| Show album art or a thumbnail sent by the PC | `BIND_SET` kinds are 0 int, 1 bool, 2 text. Images exist only inside the package (24 KB in total) | `proto/spec.md` 0x30; `docs/PLAN.md` planned an "image ref" kind |
| Toggle the play/pause button's icon | A button's `icon` and `text` are static. `bind` on a button has no meaning | `docs/packages.md` |
| Take screenshots, declared to the user | The permission list is `keys`, `media`, `system-stats`, `network`, `files`, `programs`. None covers screen capture | `docs/registry.md` |
| Show non-ASCII song and video titles | Text fonts hold printable ASCII plus `° · × – — ’ • … ‹ ›` only. Japanese, Chinese, Arabic, accented Latin and so on show as blanks | `docs/packages.md` (fonts) |

## Requests, in priority order

### 1. Runtime screen switching (must have)

Morpheus is pointless without this. Two possible shapes. fenixOS should pick one, or offer something better:

- **a. A `view` element** (or `pages`/`switch`): a container that shows exactly one of its children, chosen by an int binding (`"bind": "mode"`, value = child index). Children that are not shown are not laid out. This fits the declarative model best, and the board can animate the change the way it animates other value changes (DESIGN.md: apps have no motion props, the board animates).
- **b. Bindable visibility:** a prop like `"show": "<binding>"` on any element, which takes a bool binding.

Morpheus would have about 4–6 modes in one package. That stays inside 200 elements and 64 bindings if the modes share parts.

Questions for fenixOS:
- Do hidden children still count against RAM while the app is open, or are they built only when shown?
- Should values bound inside a hidden child be kept, the way the per-binding cache already works?

### 2. Image binding (must have for media)

A new `BIND_SET` kind (3, image) sets an `image` element's pixels at runtime.

- **Use:** album art from SMTC, about 96×96, maybe 120×120 in landscape.
- **Format:** RGB565 raw is enough. 96×96 is 18 KB, about 0.2 s over USB at 921,600 baud. It would be sent once per track, not per frame.
- **Possible rules:** the element declares its `w` × `h` and a `bind`. The PC sends exactly that size, and the SDK scales and converts. Cap the size per binding and per app. Cache it like the other bindings while the app is open.
- **SDK:** something like `app.set_image(bind, w, h, &rgb565)` or `app.set_image_png(bind, &png)`, where the SDK does the resizing.
- **Message size:** a sidecar message is at most 64 KB, which is enough for this.

Questions for fenixOS:
- Is there heap or PSRAM for one or two such buffers while an app is open? (The CYD has no PSRAM, so this matters.)
- Should it be limited to one image binding per app?

### 3. Bindable icon and button text (should have)

A play/pause button needs its icon to follow the playback state. Ideas:

- `bind` on a `button` sets its text (string), or
- a separate binding for the icon (an index into the 20 built-in glyphs, or a name).

Also useful: the bound `icon` element taking its glyph from a binding (for example a mute or unmute mic).

Built-in glyphs Morpheus would need beyond today's 20 (Phosphor fill): `play`, `pause`, `skip-forward`, `skip-back`, `speaker-high`, `speaker-x`, `microphone`, `microphone-slash`, `headphones`, `game-controller`, `music-notes`, `video`, `phone`, `sparkle`. If they are not added, Morpheus can ship them as A8 `image` assets (within the 24 KB), but then they cannot switch at runtime unless request 1 or 3 covers images too.

### 4. A `screen` permission (must have, registry only)

Morpheus takes screenshots (the whole display or the foreground window) and sends them to an AI API. AGENTS.md says: "do no more than declared". Add a permission such as `screen` ("Captures your screen") to the registry rules (`tools/registry.py`), the Add app permissions sheet and the app template's `tools/app.py`. Morpheus will capture only through Windows Graphics Capture: no hooks, no injection, no reading game memory.

### 5. Wider text coverage (nice to have)

Song and video titles are often non-ASCII. Options:

- Add Latin-1 / Latin Extended-A to the text fonts (covers most European titles), or
- document that apps must transliterate, and maybe ship a helper in `fenix-sdk` that maps text to the board's glyph set (`é` becomes `e`, unknown characters become `?` instead of a blank).

CJK is likely too big for flash. Say so if it is.

### 6. Nice to have

- **Bindable `min`/`max` on `slider` and `bar`.** Without it, Morpheus scales the seek bar to 0–1000. That works, so this is low priority.
- **Shorter `opened` resync.** Morpheus may need to send about 50 values when it opens. Confirm that is fine over the link, or suggest a batch `set`.
- **Label overflow.** Track and video titles are long. What does a bound label do when its text is wider than the label: wrap, clip, or ellipsis? Morpheus would like a `max_lines` or an ellipsis option (`…` is already in the fonts).

## Questions (please answer)

1. **Settings storage:** Morpheus keeps the user's DeepSeek API key in a `text` setting. Where does the PC app store settings, is it encrypted (DPAPI?), and does it show up in logs or exports? If it is plain text, could a `secret` setting type (masked, stored with DPAPI) be added?
2. **Elevated helper:** Morpheus will set `pc.elevated: true` (UAC once each start, for ETW frame timing to get FPS). The docs say non-elevated helpers run in a job object. How does fenixOS make sure an elevated helper ends: only `stop` plus the closed pipe? And are crash restarts also elevated (another UAC prompt each time)?
3. **Permissions Morpheus will declare:** `media`, `keys` (Discord mute/deafen keybinds), `system-stats`, `network` (DeepSeek), and the new `screen`. Is that list right for these uses?

## How Morpheus plans to use these

- One `view` with 6 children: setup, game, call, video, media and glance (`screens.md` in the Morpheus repo has the layouts). About 145 elements and 48 bindings in total.
- One image binding: 80 × 80 RGB565 album art or video thumbnail, sent once per track.

## What Morpheus does in the meantime

Until requests 1 and 2 ship, Morpheus can only be one generic card whose text, bar and buttons change with the context. Once fenixOS ships them, Morpheus sets `min_fenixos` to that release.

# Morpheus: board screens

Design for the board half. It follows `docs/fenixos/design.md` (locked): dark, `surface_1` cards with radius 12, `label` / `label_2` text, one accent, Inter, Phosphor fill icons, touch targets at least 40 px. Areas: landscape **320 × 208** (12 px margin, so 296 wide), portrait **240 × 288** (10 px margin, so 220 wide). Anything below the fold scrolls.

These screens assume the fenixOS features in `fenixos-requests.md`: a `view` element that shows one child picked by a binding, an image binding for art, and built-in glyphs for play, pause and so on. **Without them** see "Fallback" at the end.

## Principles

1. **One glance, one job.** Each mode puts its most useful thing first and biggest: the track, the score, the call timer. Secondary controls sit below the fold.
2. **Always say why.** Each mode's first line names the source in `caption_medium` / `label_2`, for example "Spotify", "VALORANT · Ascent", "Discord · Squad voice", "Brave · github.com". The user always knows what Morpheus thinks is happening.
3. **Live values come from Windows, words come from the AI.** Seek position, volume, timers and FPS update every second. AI text carries its age ("4 s ago") so stale reads are obvious.
4. **Shared bindings.** `volume`, `mic` and `deafen` have one name each, so a change shows in every mode at once.
5. **Calm morphing.** A mode changes only after the new context has held for 3 s. The board animates the change itself. A "Keep this view" switch pins the current mode.

## Which mode shows

Highest priority first. The PC sets `mode` (an int):

| `mode` | View | When |
|---|---|---|
| 0 | Setup | No DeepSeek key yet. Media, game and call still work without one. |
| 1 | Game | A full-screen game is in front. Call and media controls move into it. |
| 2 | Call | A Discord, Teams or Zoom call is active and no game is in front. |
| 3 | Video | Media is playing and the source is a browser or video player. |
| 4 | Media | Media is playing (music). Stays for 60 s after pause. |
| 5 | Glance | Anything else: the AI's read of the screen. |

If "Keep this view" is on, `mode` stays where it is until it is switched off.

## Media (mode 4)

```
Landscape 296 wide                         Portrait 220 wide
+------------------------------------+     +----------------------------+
| +------+ Spotify                   |     | +------+ Spotify           |
| |      | Midnight City             |     | |      | Midnight City     |
| | art  | M83                       |     | | art  | M83               |
| | 80px | 1:12 of 4:03              |     | +------+ 1:12 of 4:03      |
| +------+                           |     | [========o--------------]  |
| (<<) (>||) (>>)  [=====o---------] |     |     (<<)  (>||)  (>>)      |
+------------------------------------+     +----------------------------+
+------------------------------------+     +----------------------------+
| spk  [==========o---------]        |     | spk [=========o-------]    |
+------------------------------------+     +----------------------------+
--- fold ---                               (everything fits, 264 of 268)
| sq  Keep this view           (o ) |
```

- **Card 1** (`surface_1`, pad 10, gap 8):
  - **Top row:** `image` 80 × 80 bound to `art`, then a column of: `caption_medium` `label_2` source, `title` track, `label` `label_2` artist, `caption` `label_2` "1:12 of 4:03".
  - **Transport row:** previous and next are icon buttons on `surface_2`. Play/pause is the one primary (`accent_fill`). The seek `slider` (0–1000, bound to `pos`, `event: seek`) takes `grow: 1`. In portrait the transport row is a column: the slider first at full width, then the buttons centred.
- **Card 2:** control row with a speaker icon (`label_2`), a volume `slider` (`grow: 1`, bound to `volume`) and nothing else.
- **Height:** landscape about 148 + 8 + 60. The volume card is partly below the fold, which is fine. Portrait fits completely.

## Video (mode 3)

Same layout as Media, with three changes:

- **Art:** the video thumbnail, centre-cropped to 80 × 80 by the PC half.
- **Source line:** the site and the browser ("YouTube · Brave").
- **Extra card below the fold:** "About this video", 2–3 short AI lines from the title and a screenshot. It fills in only after the first glance call, and shows "--" until then.

## Game (mode 1)

```
Landscape                                    Portrait
+--------------------------------------+     +----------------------------+
| ctrl VALORANT · Ascent        4 s ago|     | ctrl VALORANT       4 s ago|
|  Your team     Round      Enemy      |     | You      Round      Enemy  |
|     7           13          5        |     |  7        13          5    |
|  Enemy likely on eco: take B fast    |     | Enemy likely on eco:       |
+--------------------------------------+     | take B fast                |
--- fold ---                                 +----------------------------+
+--------------------------------------+     +----------------------------+
| Credits                        3,900 |     | Credits              3,900 |
| ------------------------------------ |     | -------------------------- |
| Enemy buy                  Likely eco|     | Enemy buy       Likely eco |
| Spike                       Planted B|     | Spike            Planted B |
| Ults ready                    2 of 5 |     | Ults ready          2 of 5 |
+--------------------------------------+     +----------------------------+
+--------+ +--------+ +--------+             +-------+ +-------+ +-------+
| FPS    | | GPU    | | CPU    |             | FPS   | | GPU   | | CPU   |
| 238    | | 71 %   | | 34 %   |             | 238   | | 71 %  | | 34 %  |
| [====] | | [===]  | | [==]   |             | [===] | | [==]  | | [=]   |
+--------+ +--------+ +--------+             +-------+ +-------+ +-------+
+--------------------------------------+
| mic  Microphone                (o ) |   control rows with switches,
| hp   Sound                     (o ) |   shared bindings mic / deafen
| ♪    Midnight City · M83   (>||)    |   only when music is playing
| spk  [==========o---------]          |
+--------------------------------------+
| sq  Keep this view            (o ) |
```

- **Hero card:**
  - **Header row:** game icon, then `caption_medium` source ("VALORANT · Ascent"), then `caption` age on the right.
  - **Score row:** `space_between`, three columns: a `label` / `label_2` label above each `clock_56` number (us, enemy) and a `title_l` round in the middle.
  - **The call:** one `body` line, the AI's tactical read, wrapping to 2 lines at most.
- **Match read card:** 4 label/value rows with separators. The labels are **bindings too**, so the AI picks the four facts that matter for whichever game is running. For Valorant: credits, enemy buy, spike, ults. For a game it does not know, it might be kills, objective, timer.
- **Stats:** a row of 3 value cards (FPS from ETW, GPU from NVML/ADLX, CPU from `system-stats`), each with a 6 px bar. It stays a row in portrait, because the cards are short.
- **Quick controls card:** microphone and sound switches (Discord keybinds), a now-playing row when music is on (track text plus a play/pause icon button), and volume.
- **Not in a game, but a game window is in front:** the hero shows the game name and session time, and the AI read starts once a match is visible.

## Call (mode 2)

```
Landscape                                       Portrait
+-------------------+ +----------------------+  +----------------------------+
| ph Discord        | | mic Microphone  (o ) |  | ph Discord · Squad voice   |
|    Squad voice    | | ------------------   |  |        12:04               |
|   12:04           | | hp  Sound       (o ) |  | mic Mic on                 |
| mic Mic on        | |                      |  +----------------------------+
+-------------------+ +----------------------+  +----------------------------+
+--------------------------------------------+  | mic Microphone       (o )  |
| spk [==========o---------]   Call volume   |  | hp  Sound            (o )  |
+--------------------------------------------+  +----------------------------+
```

- **Hero:**
  - **Source:** the app and channel.
  - **Timer:** the call time in `clock_56`, set once a second.
  - **Mic status:** an icon and words ("Mic on" / "Muted"), never a lone dot.
- **Controls:** switches for microphone and sound. The switch colour `success` means on. In landscape the two cards sit side by side (card row pattern). In portrait they stack.
- **Below the fold:** call volume, which is the volume of Discord's own audio session, and, when Discord is visible on screen, "Speaking: alex, sam" read by the AI.

## Glance (mode 5)

```
Landscape                                   Portrait
+------------------------------------+      +----------------------------+
| spark Brave · github.com   20 s ago|      | spark Brave       20 s ago |
| Reading the fenixOS registry docs  |      | Reading the fenixOS        |
| • Apps list from registry/index    |      | registry docs              |
| • Downloads must be tariqaq repos  |      | • Apps list from registry  |
| • Next: open a PR with the json    |      | • Downloads: tariqaq only  |
+------------------------------------+      | • Next: open a PR          |
(       Look it up        )  surface_2      +----------------------------+
--- fold ---                                (      Look it up      )
| CPU 34 %   RAM 61 %   GPU 12 %      |
| sq  Keep this view           (o )  |
```

- **Card:**
  - **Header:** a sparkle icon, the source ("Brave · github.com") and the age.
  - **Title:** the AI's one-line "what you are doing", in `title`, wrapping to 2 lines at most.
  - **Lines:** up to 3 bullet lines (`body`, each about 36 characters or fewer), bound to `g1` `g2` `g3`.
- **Action:** one secondary button, "Look it up" (`event: look`). It runs a web search on the current context and replaces the lines with what it finds. It costs one search ($0.00075).
- **States that reuse this layout** (text only):
  - Spend cap reached: "AI paused for today", "Media, game and call still work."
  - Offline: "Can't reach DeepSeek".
  - Locked screen: "Away".

## Setup (mode 0)

One card: the `info` icon, `title` "Add your DeepSeek key", and `body` `label_2` "Open Morpheus in the fenixOS app on your PC and paste it in Settings." No button: the board cannot open the PC page.

## Footer (all modes, below the fold)

A control row: the `squares-four` icon, a "Keep this view" label and a `switch` bound to `pin` (`event: pin`).

## Budget

| | Estimate | Limit |
|---|---|---|
| Elements | ~145 | 200 |
| Bindings (shared names) | ~48 | 64 |
| Images | 10 glyphs of 20 × 20 A8 = 4 KB, plus art 80 × 80 RGB565 = 12.5 KB, sent at runtime | 24 KB packaged |
| Depth | 6 (root, view, mode, card, row, column) | 8 |

## Fallback without the fenixOS features

If fenixOS cannot add `view` and the image binding, use one generic layout for every mode:

- **Main card:** source line, title, subtitle, a progress `slider` and 4 label/value rows.
- **Buttons:** three text buttons whose meaning follows the mode ("Prev", "Play / pause", "Next"; or "Mic", "Sound", "Look it up").
- **Below the fold:** the volume row and the stats row.

It changes words, not shape, and has no art. It works today, but it is not really Morpheus.

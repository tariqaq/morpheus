# Morpheus: ideas

Working notes. Nothing here is decided until it is in **Decisions** at the bottom.

## The idea

Morpheus is a fenixOS app. The user switches it on in the fenixOS PC app, it lands on the board's tiles, and they open it on the board.

On the board it is one dashboard that **morphs to match what the user is doing on the PC**:

- **Listening to music or watching media:** it becomes a live media player: album art, title and artist, play/pause and skip, a live seek bar, volume.
- **Watching YouTube or other video:** same player, shaped for video.
- **Playing a game (say Valorant):** it shows game context worked out from what is on screen.
- **And so on** for other activities.

The PC half does all the work. It works out the context, asks an AI model when it needs to, and pushes the result to the board. The AI is DeepSeek (`deepseek-flash`, which is DeepSeek-V4.1-Flash-1017). It takes images, so it can look at screenshots. DeepSeek also offers web search.

Elevation is available: the PC half may ask for UAC (`pc.elevated: true`) if it needs admin.

## How it could work

```
PC half (Windows, Rust, fenix-sdk)
  1. Detect     what the user is doing (cheap signals first, AI only when needed)
  2. Decide     which mode the board should be in
  3. Fill       the mode's values (title, progress, lines of text, ...)
  4. Push       app.set(...) to the board; handle taps (play/pause, volume, ...)
Board half (declarative layout)
  one screen per mode, already in the package; the PC picks which one shows
```

The AI does not draw UI. The board can only show elements that are already in the package. So the AI picks a **mode** and fills that mode's **slots** (short text, numbers). The output is JSON, which DeepSeek supports (`response_format: json_object`).

## What we checked

### DeepSeek API (api-docs.deepseek.com, read 2026-11-01)

| Item | What the docs say |
|---|---|
| Model ids | `deepseek-flash` (version DeepSeek-V4.1-Flash-1017) and `deepseek-v4-pro` (V4-Pro-0813). `deepseek-chat` and `deepseek-reasoner` were retired 2026-07-24. |
| Images | `deepseek-flash` only (Pro has no vision). JPEG/PNG/GIF/WebP, as base64 data URL, http(s) URL or `file_id`. `detail`: `low` (512×512), `high`, `original`, `auto`. Each image costs at most 1024 tokens. Images are allowed in `user` messages only. |
| Thinking | On by default. `reasoning_effort`: `none`, `low`, `high` (default), `max`. Thinking adds `reasoning_content` and output tokens, which add cost and delay. |
| Web search | **Not** a chat tool. Separate endpoints: `POST /v1/tools/search` and `/v1/tools/fetch`, **$0.75 per 1,000 successful requests** each. Billing started 2026-11-01. Our own function tools can wrap them. |
| JSON / tools / streaming | JSON mode yes (the prompt must say "json"). Function calling yes. Streaming yes. |
| Price, `deepseek-flash`, per 1M tokens | Input cache miss $0.15 off-peak / $0.30 peak. Cache hit $0.003 / $0.006. Output $0.60 / $1.20. |
| Peak hours | 01:00–04:00 and 06:00–10:00 UTC, Mon–Fri. In GMT+8 that is **09:00–12:00 and 14:00–18:00**. |
| Limits | Context 1M. Concurrency 2500 per account. |
| Data | The API docs say nothing about where data is stored. Third-party write-ups quote DeepSeek's privacy policy as storing data in the PRC ([VentureBeat](https://venturebeat.com/ai/is-deepseek-really-sending-data-to-china-lets-decode/), [TechRadar](https://www.techradar.com/computing/cyber-security/is-deepseek-ai-safe-or-is-it-just-a-data-minefield-waiting-to-blow-up)). Not yet read from the policy itself. |

**Rough cost per call** (one screenshot at ≤1024 tokens, ~300 tokens of prompt, ~150 tokens of JSON out, thinking `none`): about **$0.0003 off-peak, $0.0006 peak**. Thinking `low` or `high` adds roughly $0.0003–0.002 a call.

| Pattern | Calls a day (8 h use) | Cost a day |
|---|---|---|
| Screenshot every 5 s | ~5,800 | ~$1.6–3.3 |
| Only when the context changes | ~100–300 | ~$0.03–0.2 |
| Text only (window title, no image), on change | ~100–300 | under $0.05 |

The current balance is $2.62. Periodic screenshots would use it up in about a day. Calling only on change would last months.

### fenixOS board (docs/packages.md, proto/spec.md on fenixOS `main`; latest tag v0.1.19)

These are the hard parts. The current package format **cannot morph**:

1. **`hidden` is static.** It is a compile-time prop (`hidden: 0/1`), not bindable. The PC cannot show or hide parts of the layout at runtime, so it cannot switch between a media screen and a game screen.
2. **No runtime images.** `BIND_SET` kinds are int, bool and text only. Images must be inside the package (24 KB in total). So no live album art or video thumbnails. (`docs/PLAN.md` once planned an "image ref" binding kind; it was never built.)
3. **Text limits.** Strings are at most 255 bytes. Board fonts are ASCII plus a few symbols, so titles in Japanese, Chinese, Arabic and other non-ASCII scripts show as blanks unless we transliterate them.
4. **Size caps for all modes together:** 200 elements, 64 bindings, 40 KB RAM estimate. That is enough for about 4–6 modes if they share parts.
5. **Values only while open.** Values reach the board only while Morpheus is open on it. On `opened`, the PC must resend everything.
6. This template targets fenixOS v0.1.13. fenixOS is now at v0.1.19.

What works today: bound labels (text), bars and sliders (int), switches (bool), buttons and sliders that send events. A live seek bar (`slider` bound to position) and a volume slider work now.

**Fix options:**
- **A. Extend fenixOS** (it is our repo). Add a bindable visibility, or a `view`/`pages` element that shows one child picked by an int binding. Add an image binding kind (a 96×96 RGB565 cover is 18 KB, about 0.2 s over USB at 921,600 baud). This needs a firmware, `fenix-pkg` and `fenix-sdk` release, and Morpheus would need `min_fenixos` set to that release.
- **B. Stay inside today's format.** One generic layout (a big title, a subtitle, a progress bar, 3 lines of text, a row of buttons) whose text changes per context. It works now, but it does not really morph and has no artwork.

### Windows side: working out the context

Most contexts need **no AI**. Windows already reports them:

| Signal | Windows API | Gives | Notes |
|---|---|---|---|
| Media | System Media Transport Controls (`GlobalSystemMediaTransportControlsSessionManager`, WinRT) | Title, artist, album, **album art**, position and length, playing or paused, and play/pause/next/prev/seek | Covers Spotify, browsers (YouTube in Brave/Chrome/Edge), most players. Permission: `media`. |
| Volume | Core Audio (`IAudioEndpointVolume`) | Master volume and mute, read and set | `media` |
| Foreground app | `GetForegroundWindow`, window title, its process's exe name | "VALORANT-Win64-Shipping.exe", "Visual Studio Code", a tab title | Asks about one window only, which is not scanning processes. See the rules below. |
| Full screen or game | `SHQueryUserNotificationState` | D3D full screen, presentation mode, busy | Cheap "a game is running" hint |
| Screen | Windows Graphics Capture | Screenshot of a window or monitor | The only signal that needs AI to read it. |

None of these need admin. Elevation would only help to read details of elevated windows, and it costs a UAC prompt every time the app starts. Keep it as an option, not a default.

### Rules and risks

- **AGENTS.md rule 6:** no hooks, injection, process scanning or input capture, and do no more than the declared permissions. The permission list is `keys`, `media`, `system-stats`, `network`, `files`, `programs`. **It has no screen-capture permission.** Taking screenshots would mean adding one to fenixOS (for example `screen`) so the user sees it before Add.
- **Anti-cheat (Valorant / Vanguard):** capturing the screen through Windows Graphics Capture is what OBS and Discord do, so it is low risk. Showing information that gives an in-game edge could break Riot's terms. Sources are weak ([Riot DevRel on Vanguard](https://www.riotgames.com/en/DevRel/vanguard), [Mobalytics](https://mobalytics.gg/?p=69932)); check Riot's developer policy before building a game mode. Never read game memory or files.
- **Privacy:** screenshots can hold passwords, bank pages and private chats, and they would go to DeepSeek. Ideas: screenshots off by default, a block list (banking, password managers, incognito windows), only the foreground window, `detail: low`, and a setting that shows what was sent.
- **Latency:** an AI call takes seconds, so AI cannot drive anything live (seek bar, timers). Live values come from Windows APIs. AI only picks the mode and writes the slow text.

## Feasibility

**Yes, with two conditions.**

1. The board must learn to switch screens (bindable visibility or a `view` element) and, for album art, take a runtime image. That is fenixOS work (option A). Without it, Morpheus is a single generic card that changes its text (option B).
2. Most of the "smart" part should be plain Windows signals, not AI. Media, volume and the foreground app are free, instant and exact. AI is worth it for the cases those cannot cover: working out what an unknown window is, summarising what is on screen, and pulling extra info from the web. This keeps the cost to cents a day and keeps screenshots rare.

## Candidate modes

| Mode | Triggered by | Shows | Needs AI? |
|---|---|---|---|
| Media | SMTC session playing | Art, title, artist, seek, play/pause, skip, volume | No |
| Video | SMTC + browser/player in front | Title, channel, seek, play/pause, volume | No (AI could add a summary) |
| Game | Full-screen D3D + known exe | Game name, session time, CPU/GPU temps, mic mute? | Optional |
| Call | Discord/Teams/Zoom in front or using the mic | Mute, deafen, call time | No |
| Work / coding | IDE or terminal in front | Focus timer, git branch? build state? | Optional |
| Unknown | None of the above | AI "glance": what it thinks you are doing plus 2–3 short lines | Yes |
| Idle | No input or screen locked | Clock, stats | No |

## Open questions

- Extend fenixOS (A) or stay inside today's format (B)? This waits on the fenixOS session's answer to `fenixos-requests.md`.
- Where does fenixOS store settings, and is the API key safe there? Asked in `fenixos-requests.md`.
- Which games get a dedicated match read beyond Valorant?

## Decisions

| # | Date | Decision |
|---|---|---|
| 1 | 2026-11-01 | **AI role: hybrid.** Windows APIs handle media, volume, the foreground app and games (free, instant). DeepSeek is called only on a context change, for windows those cannot explain, and to write the glance text. |
| 2 | 2026-11-01 | **Screenshots: allowed** (personal project, DeepSeek trusted). Whole display or foreground window, whichever works better. They must be **efficient** so credits last. Block list: incognito/private windows, password managers, and similar. |
| 3 | 2026-11-01 | **v1 modes:** media/video, game, call, AI glance (unknown). |
| 4 | 2026-11-01 | **Board morphing:** pending. The fenixOS changes are written up in `fenixos-requests.md` for the fenixOS session. Choose A or B once it answers. |
| 5 | 2026-11-01 | **Game mode shows** PC stats, session info (game, time played, current track), an AI match read (score, round, agent, map read off the HUD from screenshots) and quick controls (Discord mute/deafen, volume, media). The user's position: Morpheus only reads what the user already sees, the way OBS records it, and **never reads game memory or files**. Changed the same day: the match read **may** make tactical calls from anything the game shows (minimap, scoreboard, enemy economy). Riot's developer policy covers apps that use Riot's API. Morpheus uses no Riot API, and no rule against reading the screen was found ([Riot VALORANT developer policy](https://support-developer.riotgames.com/hc/en-us/articles/22698769097107-VALORANT), [Riot's stance on third-party apps](https://support.riotgames.com/en-us/valorant/account/third-party-applications)). The risk is the user's to take. |
| 6 | 2026-11-01 | **API key:** a `text` setting on the app's page in the fenixOS PC app. |
| 7 | 2026-11-01 | **User settings:** DeepSeek API key, daily spend cap (USD), screenshot scope (foreground window or whole display), and extra block-list words. Thinking effort is **not** a setting: Morpheus picks it per call. |
| 8 | 2026-11-01 | **Web search:** only when useful. The AI calls it through a function tool, and it counts toward the spend cap. |
| 9 | 2026-11-01 | **Elevation:** `pc.elevated: true`. Windows asks for UAC once each time fenixOS starts the helper (when the app is switched on, and when fenixOS starts). Admin is used for ETW frame timing, which gives FPS without touching the game. |
| 10 | 2026-11-01 | **Discord mute/deafen: keybinds** sent with `SendInput` (`keys` permission). Discord's RPC voice control needs scopes that Discord approves per app, so it is not an option for a personal app. Morpheus cannot read Discord's mute state, so the board shows the state it last set. The screenshot read can correct it when Discord is visible. The user sets their Discord keybinds in Morpheus settings. |
| 11 | 2026-11-01 | **Identity:** name Morpheus, id `io.github.tariqaq.morpheus`, tile `purple`, icon Phosphor `shapes` (forms changing shape, which is what the app does). The board tile `app/board/icon.png` is the same glyph, white on transparent. |
| 12 | 2026-11-01 | **Screens:** open creative brief inside the locked design language. They must be adaptive and intuitive. Designed in `screens.md`. |

### How the game and call controls could work (to check)

- **Discord mute/deafen:** Discord's local RPC voice control needs scopes that Discord approves per app. The simpler path is to send the user's Discord keybinds (for example Ctrl+Shift+M) with `SendInput`, which needs the `keys` permission. Teams, Zoom and Meet have their own mute shortcuts. Sending keys is output, not input capture, so rule 6 allows it.
- **FPS:** reading FPS without hooking a game means ETW (PresentMon's method). That needs admin or the Performance Log Users group, which would be a good use of `pc.elevated: true`. Alternatively, leave FPS out.
- **GPU load and temperature:** NVML (NVIDIA) and ADLX (AMD) need no admin.
- **CPU temperature:** avoid. Common tools read it through the WinRing0 kernel driver, which Vanguard and other anti-cheats block or flag. Show CPU load instead (`system-stats`).

### Keeping screenshots cheap (follows from decision 2)

- Capture only when something changes: the foreground app or window title, the SMTC track, or full-screen state. Use a slow fallback timer (say 60 s) for when nothing else fires.
- Skip the upload when the new frame looks like the last one (a small perceptual hash diff).
- Downscale on the PC before sending and use `detail: low`. This keeps each image near the 1024-token cap or below.
- Use thinking `none` or `low` for the classifier call. Use `high` only for the glance text, and only when the result is shown.
- Cache by app plus window title, so the same context is not asked twice.
- Put a fixed system prompt first so DeepSeek's cache hits ($0.003 per 1M instead of $0.15).
- Add a daily spend cap setting. The PC half counts the `usage` tokens from each reply and stops AI calls at the cap.

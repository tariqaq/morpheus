# Morpheus

A **fenixOS** app (ESP32-2432S028R board and its Windows app; `tariqaq/fenixOS`). Its board screen morphs to match what you are doing on the PC: a media player while music or video plays, a match read and quick controls while you game, call controls on Discord, and an AI glance for anything else. The PC half works out the context from Windows (media session, volume, foreground window) and DeepSeek reading screenshots.

**Status: planning.** The code is still the template's starter app (a counter). See `ideas.md` for the design and decisions, `screens.md` for the board screens, and `fenixos-requests.md` for the fenixOS features Morpheus needs.

The rest of this README is the fenixOS app template's guide, made for fenixOS **v0.1.13**.

## Start a new app

1. Copy this folder into a new repo under `github.com/tariqaq/` (the registry only accepts downloads from there).
2. Pick an id `io.github.tariqaq.<name>` and set it, with `name`, `color` and `version`, in `manifest.json` and `app/board/app.toml`. Set `repo` and `description` in `manifest.json`. Replace `app/board/icon.png` (white glyph on transparent).
3. Open Claude Code in the repo and describe the app. `AGENTS.md` tells it what to read and the rules; it builds the board screen in `app/board/layout.json` and, if needed, the PC half in `app/pc`.

No PC half needed? Delete `app/pc`, the root `Cargo.toml`, and `"pc"` in `manifest.json`.

## Tools

- Rust (stable) and Python 3.11+.
- `fenix-pkg`, the package compiler: `cargo install --git https://github.com/tariqaq/fenixOS --tag v0.1.13 fenix-pkg`.
- fenixOS installed, with the board plugged in, to test.

## Build, test, release

```powershell
pwsh tools/build.ps1        # dist/app.fxp, dist/app.exe; checks the manifest
python tools/app.py local   # dist/registry for FENIX_REGISTRY (docs/fenixos/registry.md)
```

Release: bump `version` in both files, tag `v<version>`, push the tag. The workflow attaches the files to the release and uploads the filled manifest; add it to `tariqaq/fenixOS` as `registry/apps/<id>.json` by PR (steps in `docs/fenixos/registry.md`).

## Layout

| Path | What |
|---|---|
| `AGENTS.md`, `CLAUDE.md` | Rules for AI agents |
| `docs/fenixos/` | App format, design language, PC half, registry (copies from fenixOS) |
| `design/tokens.json` | Design tokens (copy) |
| `manifest.json` | Registry manifest: name, colour, permissions, settings |
| `app/board/` | Board half: `app.toml`, `layout.json`, `icon.png` |
| `app/pc/` | PC half (Rust, `fenix-sdk`) |
| `tools/` | `build.ps1`, `app.py` |
| `.github/workflows/release.yml` | Tag to release |

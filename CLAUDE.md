# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**Death to the False Gods** is a text-heavy fantasy game. Its two pillars:

- **Deep world generation**: procedurally generated history, geography, peoples, and lore, with the world presented primarily through prose.
- **Deep combat simulation**: bodies are modeled as layered anatomy (flesh, bone, organs, nervous system), not stats or hit points. Injury, impairment, and death emerge from what was physically damaged (a severed nerve disables a limb, a punctured lung impairs breathing). There is no HP pool. Don't introduce one, even as a convenience.

The repo is still the Tauri + SvelteKit starter template (the `greet` command and demo page). No game code exists yet.

## Commands

Package manager is **bun** (Tauri's `beforeDevCommand`/`beforeBuildCommand` call `bun run …`).

| Task                                                      | Command                                                |
| --------------------------------------------------------- | ------------------------------------------------------ |
| Run the desktop app (Vite + Rust, hot reload)             | `bun run tauri dev`                                    |
| Frontend only in a browser (no Rust; `invoke` calls fail) | `bun run dev`                                          |
| Production desktop bundle                                 | `bun run tauri build`                                  |
| Type-check Svelte/TS                                      | `bun run check`                                        |
| Format check / fix                                        | `bun run lint` / `bun run format`                      |
| Rust check / tests                                        | `cargo check` / `cargo test` (run inside `src-tauri/`) |
| Single Rust test                                          | `cargo test <test_name>` (inside `src-tauri/`)         |
| Add a shadcn-svelte component                             | `bunx shadcn-svelte@latest add <component>`            |

No JS test runner is configured yet.

## Architecture

Two halves talk over Tauri IPC:

- **`src-tauri/` (Rust)**: the native backend. `src/lib.rs` holds `run()` and registers commands in `tauri::generate_handler![...]`. `main.rs` only calls `run()`. A new `#[tauri::command]` does nothing until it's added to that macro list. Plugins also need their permissions added to `capabilities/default.json`.
- **`src/` (SvelteKit + Svelte 5)**: the UI. It calls Rust with `invoke('command_name', { args })` from `@tauri-apps/api/core`. Argument keys are camelCase on the JS side and map to snake_case Rust parameters.

Constraints that span several files:

- **SPA only.** `+layout.ts` sets `ssr = false`, and `adapter-static` builds to `build/` with an `index.html` fallback, which Tauri loads as `frontendDist`. There is no Node server, so no `+page.server.ts`, `+server.ts`, or form actions. Anything server-like belongs in Rust.
- **Fixed dev port.** Vite must serve on `1420` (`strictPort`) because `tauri.conf.json` points `devUrl` there. Vite ignores `src-tauri/`, and Cargo handles Rust rebuilds.
- **Svelte 5 runes** (`$state`, `$props`, `$derived`, `{@render}`). Don't use legacy `export let` / `$:` syntax.

## UI and styling

- Tailwind v4 is configured in CSS, not JS. Theme tokens (oklch colors, `--radius: 0rem`) live in `src/routes/layout.css`, which is also the Tailwind entry and the stylesheet for `prettier-plugin-tailwindcss`. Dark mode is class-based (`.dark` ancestor).
- shadcn-svelte (`components.json`) uses style `luma`, base color `stone`, and lucide icons. Generated components go to `$lib/components/ui`. Use the `cn()` helper from `$lib/utils`.
- Import with the `$lib` alias.

## Formatting

Prettier settings: tabs, single quotes, no trailing commas, print width 100, plus the Svelte and Tailwind class-sorting plugins. `static/` and lockfiles are ignored.

## Agent skills

### Issue tracker

Issues live in this repo's GitHub Issues (gh CLI). See `docs/agents/issue-tracker.md`.

### Triage labels

Default five-role vocabulary (needs-triage, needs-info, ready-for-agent, ready-for-human, wontfix). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one root `CONTEXT.md` + `docs/adr/`. See `docs/agents/domain.md`.

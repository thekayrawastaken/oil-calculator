# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file web app: `oil calculator.html`. A planner/optimizer for an idle game ("oil" / gas production). No build, no package manager, no tests. Open the HTML file directly in a browser to run it.

## Architecture

Everything lives in one HTML file with three `<script>` blocks:

1. **`<script id="embeddedState">`** (line ~554) — sets `window.__EMBEDDED_STATE__` with a snapshot of `zones`, `catalog`, `inventory`, `money`, etc. On load, `loadState()` prefers `localStorage` under key `idle-optimizer-v1`, but imports the embedded snapshot once (deduped by a hash stored in `localStorage` as `<KEY>:imported:<hash>`). To force a re-import, clear that hash key or wipe storage via the in-app Reset button.
2. **Main `<script>`** (line ~555 to end) — all app logic, organized by `// ============ SECTION ============` banners:
   - `I18N` — `I18N.en` / `I18N.tr` dictionaries + `T(key, ...args)` + `applyI18n()` (driven by `data-i18n` / `data-i18n-ph` attributes in markup).
   - `CONSTANTS` — `ZONE_MULTS = [3,5,3, 2,2,2, 1,1,1, 1,1,1]` and `SLOT_SHAPES = ['L','M','L','M','S','M','L','M','L']` together define the fixed 12-zone × 9-slot grid and which slots accept L (2×2), M (2×1), S (1×1) machines. `STORAGE_KEY = 'idle-optimizer-v1'`.
   - `STATE` — single mutable `state` object: `{ zones[12], catalog[], inventory{id:count}, money, gas, sellPrice, bonus, tacoGas, muted, lang }`. Each zone has `{mult, disabled, slots[9]}`; each slot has `{shape, machine, split, sub:[null,null]}` — an M slot can be split into two S sub-slots.
   - `HELPERS` — `iterSlots()` yields `{zoneIdx, slotIdx, sub, machineId}` and is the canonical way to walk placements. `parseNum` accepts suffixes K/M/B/T/Q; `fmt` is the inverse. `slotIncome(id, zoneMult)` = `machine.rate * zoneMult`. `totals()` sums income, **skipping disabled zones**.
   - `RENDER` — `render()` is the single rerender entry point; calls `renderHeader/Palette/Zones/Calculator` and `refreshCmpOptions`. Slot DOM uses `data-zone` / `data-slot` / `data-sub` attributes; drag-and-drop is wired in `attachDnD` and handled in `handleDrop`.
   - `SUGGESTIONS` — `enumeratePlaceMoves` / `enumerateUpgradeMoves` / `enumerateRelocations` produce candidate moves; `autoPlaceOn(simZones, extraInventory)` runs the placement algorithm on a *cloned* zone array (used both for the live grid and for what-if simulation).
   - `INVESTMENT SUGGESTION` — `buildStrategies(budget)` simulates buying combinations of machines, calls `autoPlaceOn` on a sim copy, and picks the highest-income strategy.
   - `COMPARATOR` — pairs of (machine, qty) inputs; each option is scored by re-running optimal placement and comparing income deltas.
   - `CALCULATOR` — `recalc3way` solves Gas / Money / Time given any one (uses `sellPrice * (1 + bonus)` as gas→money rate); `updateAfford` computes affordability of a target machine purchase.
   - `SOUND + EFFECTS` — small WebAudio `beep`, `burstAt`, `floatGainAt` for feedback. Muted state is persisted in `state.muted`.
   - `INIT` — wires up all DOM event listeners at the bottom.

3. **Inline `<script>` tags inside markup** — none; all logic is in the two scripts above.

## Key invariants to preserve when editing

- `ZONE_MULTS` and `SLOT_SHAPES` lengths drive both the saved-state shape and the rendered grid; changing them will invalidate existing saves (loadState tolerates a shorter saved zones array but not a mismatched shape grid).
- Machine size (`L`/`M`/`S`) must match the slot shape, except an `M` slot in `split:true` mode accepts two `S` machines in `sub[0]`/`sub[1]`.
- Inventory counts (`state.inventory[id]`) are independent of placement; `availableCount = owned - placed`. Always use `availableCount` before placing, never decrement inventory on place.
- Disabled zones are still rendered and still hold placements, but contribute 0 to `totals()` — keep this behavior when adding new aggregations.
- `saveState()` writes to `localStorage`; call it after any mutation that should persist. Most mutation paths already call `render()` which the existing code follows with `saveState()` — match that pattern.

## Running / debugging

- Open `oil calculator.html` (or `index.html`, same content) directly in a browser (no server needed).
- To reset to the embedded state: in the app click **Reset**, or in DevTools run `localStorage.clear()` and reload.
- To inspect live state: `state` is module-scoped inside the IIFE-less script, so it is accessible as a free variable from the DevTools console while the page is loaded.

## Deployment (GitHub Pages)

- Live site: https://thekayrawastaken.github.io/oil-calculator/
- Repo: https://github.com/thekayrawastaken/oil-calculator (public, owner `thekayrawastaken`)
- The deployed file is `index.html`. **Only edit `index.html`** — do not touch `oil calculator.html` (it's the original local copy and the user no longer wants it kept in sync).
- **After any edit, automatically commit and push** — the user has asked for auto-deploy, so do not wait for confirmation:
  ```
  git add -A
  git -c user.email="devilhack0@gmail.com" -c user.name="thekayrawastaken" commit -m "<short message>"
  git push
  ```
- GitHub Pages auto-rebuilds in ~30–90s after each push. `gh` CLI is at `C:\Program Files\GitHub CLI\gh.exe` and is already authenticated.
- To check build status: `& "C:\Program Files\GitHub CLI\gh.exe" api repos/thekayrawastaken/oil-calculator/pages/builds/latest`

---
name: craft-command-engine
description: >-
  Use this skill when adding or modifying user-visible commands, wiring menu
  items, implementing schema-driven dialogs, exposing actions to the CLI/MCP
  server/JSON control channel, or writing panic_hunt fuzz tests in a
  PhotoCraft-style application.
---

# Engine-First Command Architecture & Automation

In a PhotoCraft-style application, **everything is a command**. The GUI, headless CLI, JSON TCP control channel, recorded Actions, and MCP server (`rmcp`) all dispatch commands through `Session::execute(id, params)` in `crates/engine`.

## Step-by-Step Checklist: Adding a Command

### 1. Find the Canonical Command ID
- Search `crates/ui-egui/src/menu_catalog.rs` for the target menu item.
- Using the exact ID from `menu_catalog.rs` automatically lights up the menu item in the UI without manual UI wiring.
- For non-menu actions (e.g., tool gestures or sub-commands), use the same dotted camelCase convention (`layer.smartFilter.delete`) and an empty menu path `&[]`.

### 2. Implement the Algorithm in the Lowest Applicable Layer (`L0`–`L2`)
- Place pixel/imaging math in `crates/algo`, brush math in `crates/paint`, vector math in `crates/vector`, text in `crates/text`, or color math in `crates/cms` / `crates/color`.
- Algorithms must be **pure, deterministic** (hash coordinates for noise rather than unseeded RNG), **tile-aware** (parallelized over tiles with `rayon` on native targets), and **depth-agnostic** (`U8`, `U16`, `F32`).

### 3. Register the `CommandSpec` in `crates/engine`
- Create or update a focused command module `crates/engine/src/<area>_cmds.rs` exposing `pub fn specs() -> Vec<CommandSpec>` and register it in `crates/engine/src/commands.rs`.
- Populate every `CommandSpec` field (see [command-and-control-patterns.md](./references/command-and-control-patterns.md)):
  - `id`, `label`, `menu`, `shortcut`
  - `params`: concise JSON schema doc string (e.g., `'{"radius":px=4,"mode":"a|b"="a"}'`), read by agents via `command_list`. When the `params` doc is closed and lists every key the command reads, add the command ID to `STRICT` in `crates/engine/src/param_check.rs` so `Session::execute` rejects misspelled/unknown keys across all transports (`"coalesce"` and `"target"` are accepted universally).
  - `enabled`: `fn(&Session) -> Result<(), String>` returning a clear human/agent-readable disabled reason.
  - `run`: `fn(&mut Session, &Value) -> Result<Value>` that validates parameters, respects selection masks, channel/mask targets, Smart Object non-destructive filter stacks (where applicable), and layer locks, and records **one** undo step via `Session::edit`.
  - `journal`: `true` for state-mutating commands (recorded in Actions), `false` for read-only queries.

### 4. Validate Adversarial Input (Rule 9: Never Panic)
- Validate all JSON parameters before allocating or indexing: reject negative radii, `NaN`/`inf`, zero divisors, out-of-range layer IDs, or absurd canvas sizes with `Err(EngineError::BadParams { .. })`.
- Never use `unwrap()`, `expect()`, or unchecked slice indexing `[i]`.

### 5. Wire UI Dialogs (If Parameterized)
- Filter and adjustment commands use schema-driven dialogs with live on-canvas preview (`crates/ui-egui/src/filter_dialog.rs`, `dialogs.rs`, and `smart_filter_dialog.rs` for re-editing Smart Filters and per-filter Blending Options).
- All UI state must live in `crates/ui-egui/src/state.rs` (`UiState`, serializable) and use design tokens from `theme::Tokens`.

### 6. Write Comprehensive Tests & Update Parity

Run the verification suite before finishing:

```sh
cargo test -p photocraft-engine --lib --tests
cargo test -p photocraft-engine --test panic_hunt -- --ignored
cargo xtask parity
cargo clippy -p photocraft-engine -p photocraft-ui-egui --all-targets -- -D warnings
```

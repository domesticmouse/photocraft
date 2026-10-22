# CommandSpec, Execution & Automation Patterns

## 1. `CommandSpec` Structure (`crates/engine/src/commands.rs`)

```rust
pub type Run = fn(&mut Session, &Value) -> Result<Value>;
pub type Enabled = fn(&Session) -> std::result::Result<(), String>;

pub struct CommandSpec {
    pub id: &'static str,
    pub label: &'static str,
    /// Menu placement, e.g. `["Filter", "Blur"]`. Empty = not in menus.
    pub menu: &'static [&'static str],
    /// Default shortcut in egui-style notation (`Cmd+Shift+N`), mapped per platform by the UI.
    pub shortcut: Option<&'static str>,
    /// Human/agent-readable parameter description (JSON-ish).
    pub params: &'static str,
    pub enabled: Enabled,
    pub run: Run,
    /// Record in the session journal (false for queries).
    pub journal: bool,
}
```

## 2. Writing Informative `Enabled` Predicates

Instead of returning `bool`, `Enabled` returns `Result<(), String>` so both the GUI and AI agents (via MCP `command_list`) know *why* a command is unavailable:

```rust
fn has_pixel_layer(s: &Session) -> std::result::Result<(), String> {
    let l = crate::active_layer_of(s)?;
    if matches!(l.content, LayerContent::Raster(_)) {
        Ok(())
    } else {
        Err(format!(
            "active layer is {} {} layer, not a pixel layer",
            l.content.article(),
            l.content.kind_name()
        ))
    }
}
```

When `command_list` is queried, `commands::list_entry` exposes:
- `"enabled": bool`
- `"disabledReason": Option<String>`
- `"needsDocument": bool` (distinguishes "open a document first" from preconditions like "no selection")

## 3. Respecting Context Inside `run`

A pixel/filter command's `run` implementation should check:
1. **Parameter validity & strict key check:** Parse and bounds-check numbers (`is_finite()`, range limits) returning `EngineError::BadParams`. Commands listed in `STRICT` (`crates/engine/src/param_check.rs`) automatically reject any parameter key not documented in `CommandSpec::params` (except the universal `"coalesce"` and `"target"` keys).
2. **Edit target (Pixels vs. Smart Filter vs. Mask vs. Alpha Channel):** Check `is_mask_target(params)` or `channel_cmds::edits_channel(session)` first so filters/adjustments work seamlessly on layer masks, Quick Mask, and alpha channels; when targeting a Smart Object layer (`LayerContent::Smart`), append or update a non-destructive `SmartFilter` on the layer instead of rasterizing.
3. **Locks:** Call `check_pixels_unlocked(doc, layer_id)?` or `check_all_unlocked(doc, layer_id)?`.
4. **Selection Mask & Feathering:** If `doc.selection` is `Some(mask)`, blend the modified surface with the original surface weighted by the selection mask (and layer `locks.transparency`).
5. **Coalescing Interactive Edits:** Callers dragging a slider or typing pass `"coalesce": "<gesture-id>"` in `params`; `Session::execute` extracts this key so `Session::edit` merges consecutive steps with the same key into a single undo entry.

## 4. Multi-Frontend Dispatch

Because `Session::execute(&mut self, id: &str, params: Value) -> Result<Value>` is the single entry point:
- **GUI (`photocraft-ui-egui`):** Menu items, toolbar buttons, and shortcuts invoke `session.execute(id, params)`.
- **Headless CLI (`apps/photocraft-cli`):** `photocraft-cli run in.psd --cmd <id> --params '<json>' --out out.png` runs without a display server.
- **Headless JSON-RPC (`photocraft-cli rpc`, `crates/automation/src/rpc.rs`):** Persistent newline-delimited JSON-RPC over stdio without MCP framing (`engine.execute`, `doc.open`, `doc.save`, `doc.render`, `batch`; passing `index` on `doc.*` methods targets that document for a single call without changing the active document).
- **TCP Control Server (`ui-egui/src/control.rs`):** Accepts authenticated JSON-lines requests (`command.run`, `ui.inspect`, `ui.screenshot`, `ui.pointer`, `ui.gpu.simulateLoss`) to drive and visually verify a live app instance.
- **MCP Server (`crates/automation`):** Wraps `Session` (or bridges to a running GUI over TCP) with capability-restricted filesystem roots (`cap-std` via `AuthorizedWorkspace`), exposing `command_list`, `command_run`, `command_batch`, `doc_inspect`, and `doc_render_preview`.

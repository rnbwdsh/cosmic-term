# cosmic-term --drop-down feature (pr-673 branch)

## Where the code lives
- **Repository**: https://github.com/rnbwdsh/cosmic-term
- **Branch**: `pr-673`
- **Base**: `origin/master` at `117644e` (Epoch 1.4.0)
- **PR**: https://github.com/rnbwdsh/cosmic-term/pull/2

## What works
- `--drop-down` CLI flag → wlr_layer_shell surface (TOP|LEFT|RIGHT anchor, Overlay layer, OnDemand keyboard)
- Single-instance via `/proc` scanning (no PID file, no flock, no dirs/nix deps)
- SIGUSR1 toggle: 2nd instance sends `kill -USR1 <pid>`, 1st receives via `tokio::signal::unix`
- 1ms delayed initial toggle to avoid first-frame compositor transparency race
- Header bar rendered manually in `view()` (bypasses `view_main()`)
- Separate `show_headerbar_dropdown` config for dropdown vs regular header preference
- Context drawer (Settings, About, etc.) rendered as overlay on layer surface
- Right-click context menu works (popup_parent_id uses layer surface window)
- Settings page has dropdown section first (Height slider 300-1300px + Show header in dropdown)
- Cargo.toml adds only: `tokio = { features = ["sync", "signal", "time"] }`

## Key files
| File | Purpose |
|------|---------|
| `src/dropdown.rs` | `/proc` scanning, surface creation, popup_parent_id, SIGUSR1 sub, view wrapping |
| `src/main.rs` | CLI parsing, drop_down flag, TerminalMode, ToggleDropDown handler, settings |
| `src/menu.rs` | Conditional context menu (pane-maximize hidden in dropdown, header toggle) |
| `src/config.rs` | `dropdown_height: u32` (default 400), `show_headerbar_dropdown: bool` (default true) |
| `i18n/en/cosmic_term.ftl` | `dropdown`, `dropdown-height`, `show-headerbar-dropdown` strings |

## Fat to cut (things to simplify)

### 1. `TerminalMode` enum vs `Flags.drop_down`
Currently there's both:
- `Flags { drop_down: bool }` — set from CLI, used once in `init()` to build `TerminalMode`
- `App { mode: TerminalMode }` — used everywhere else

`Flags.drop_down` is only read in `init()`. It exists because `Flags` is the struct passed through `cosmic::app::run()`. Could potentially remove it and pass the flag differently, or just document why it's needed.

### 2. `TerminalMode` could become `Option<Option<window::Id>>`?
```rust
// Current:
enum TerminalMode {
    Normal,
    DropDown { window_id: Option<window::Id> },  // None=hidden, Some(id)=visible
}

// Could be:
// None = normal mode
// Some(None) = dropdown mode, hidden
// Some(Some(id)) = dropdown mode, visible
```
The `Option<Option<>>` is ugly but eliminates the enum. Not strictly better.

### 3. Duplicate destroy+recreate logic
Both `ShowHeaderBar` and `ShowHeaderBarDropdown` handlers do the same destroy-surface + recreate-surface pattern. Could extract a helper:
```rust
fn recreate_dropdown_surface(&self) -> Task<Message> { ... }
```

### 4. `show_headerbar` vs `show_headerbar_dropdown`
Two separate config keys. In normal mode, `show_headerbar` works. In dropdown mode, `show_headerbar_dropdown` works. The old `show_headerbar` setting is hidden in dropdown settings. This is intentional — users may want headers in regular windows but not the dropdown — but adds complexity.

### 5. `SettingsWindow*` message cleanup
The `SettingsWindowOpened`, `SettingsWindowCloseRequested` messages and `settings_window_id` field were removed. Verify no dead references remain.

## Known issues
- **Initial transparency glitch**: First frame of dropdown may show transparent background. The 1ms delay helps but doesn't fully fix. Related to compositor configure event timing.
- **Daemonization zombie**: The `fork()` daemonization in `main()` leaves a brief `<defunct>` zombie. Pre-existing, not dropdown-specific.
- **Context drawer on layer surface**: Works but may have occasional input focus issues vs native window.
- **No way to set dropdown position**: Wayland `xdg_shell` has no `set_position`. The layer surface anchors to TOP|LEFT|RIGHT which is correct for a quake terminal, but means it's always full-width.

## What was tried and rejected
- **Regular window + minimize toggle**: Window appears at compositor-chosen position, not top-anchored. Unminimize doesn't work on Wayland.
- **PID file + flock**: Removed in favor of `/proc` scanning. Simpler, no filesystem artifacts.
- **nix crate**: Removed in favor of `std::process::Command("kill")` + `tokio::signal`. Fewer deps, no unsafe.
- **Settings as separate window**: Reverted in favor of context_drawer overlay on the layer surface. The popup_parent_id fix made overlays work.
- **Percentage height**: Reverted in favor of pixel height (300-1300px). Layer surface API only takes pixels, no `wl_output` geometry available.

## For the next agent
1. Read this file, the PR diff, and `src/dropdown.rs`
2. Try to simplify `TerminalMode` + `Flags.drop_down` (can we remove the enum?)
3. Extract the duplicate destroy+recreate pattern
4. Test: `cargo build --release && ./target/release/cosmic-term --drop-down`
5. Test toggle: run a second `cosmic-term --drop-down` — first should hide/show
6. Test settings: open settings from hamburger menu, adjust height slider, toggle header
7. Test context menu: right-click terminal, check menu items
8. The `Cargo.toml` diff should be minimal: only `tokio` features added

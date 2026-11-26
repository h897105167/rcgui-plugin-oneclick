# Changelog

## [Unreleased] 0.4.1 — deprecated

Not tagged yet. Installing, starting and stopping rclone is RCGUI's own job, so
this plugin does none of it. The source stays here, buildable and readable, as the
reference implementation for `MIGRATION.md`; nothing is installed by default.

- `config.json` declares no `on_start` / `on_exit`. With both hooks in place the
  plugin's `auto_start` and the app core's own startup would each spawn an
  `rclone rcd`, and the two would fight over port 5572 and kill each other on
  exit.
- The `install`, `start`, `autostart`, `stop` and `shutdown` actions answer with a
  deprecation notice: a normal reply (`done`) carrying `lamp: warn` and a toast,
  not an error, because nothing failed. They never reach `proc.start` /
  `proc.stop`.
- `state`, `pick_exe`, `search_exe`, `search_deep`, `pick_history` and
  `revalidate` still work: they read, and they leave no rclone process behind.
- The install / start / stop labels end in `（已弃用）` / `(deprecated)`; the panel
  description and the plugin summary say the same thing up front.
- `config.json` declares one page, `settings` (设置 / Settings). Every control
  except the status line, progress bar and checks table carries
  `"page": "settings"`, so a host that draws pages shows one named group instead
  of an unpaged list. A host older than 0.3.0 ignores the field and lays everything
  on one page.
- The release workflow is manual only (`workflow_dispatch`): it checks the
  versions, runs format, wasm build and tests, packs, and uploads a build
  artifact. No tag trigger and no `gh release create`, so the host repository's
  bundle cannot pick up a plugin that manages rclone itself.
- `ci.yml` runs `cargo fmt --check`, `cargo test`, `cargo check --target
  wasm32-wasip1`, a `config.json` structure check and a packaging check on every
  push and pull request.
- **Packaging**: the build sets `--remap-path-prefix`, so the shipped wasm no
  longer embeds this machine's cargo path.
- `MIGRATION.md` lists the 11 settings keys the app core should keep, the
  `_state` fields worth dropping, how the core resolves a version, verifies the
  binary and builds the launch arguments, and how to confirm afterwards that only
  one rclone is running.

Nothing was deleted from the plugin's kv store. All 11 settings and the `_state`
snapshot (including the last 10 entries of executable history) are still there for
the app core — or a person — to read.

## 0.3.3

Last version that installed and managed rclone on its own.

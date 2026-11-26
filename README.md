# rcgui-plugin-oneclick

> **Deprecated (0.4.1) — built into the app.** Installing, starting and stopping rclone is RCGUI's own job now. This plugin no longer starts, stops or installs any rclone process, and no new versions will be released.
> To carry your settings and history over, follow [MIGRATION.md](MIGRATION.md). The source is kept as a reference implementation: `config.json` no longer declares `on_start` / `on_exit`, the install / start / stop buttons only return a deprecation notice, and nothing in the plugin's kv store was deleted.

**The one-click rclone plugin for RCGUI** — it used to find a local `rclone.exe`, verify it really is rclone, download a fresh one if there wasn't any, launch it as `rclone rcd`, and hand the RC endpoint back to the host. Since 0.4.1 the app core does all of that; this repository is the reference for how it was done.

English | [简体中文](README.zh-CN.md)

> Host repository: [h897105167/rclone_gui_windows](https://github.com/h897105167/rclone_gui_windows)
> Plugin authoring guide: [docs/PLUGIN.md](https://github.com/h897105167/rclone_gui_windows/blob/main/docs/PLUGIN.md)

This repository holds the plugin only: Rust source, `config.json`, packaging scripts, release CI. Every control you see in the panel is declared in `config.json` — the host draws the form.

The plugin is `"builtin": true` and `"platform": "windows"`: it ships inside RCGUI's own assets and is never fetched through `plugins/manifest.json`. It is also the one plugin allowed to make assumptions about Windows, because rclone's official builds for this flow are Windows binaries.

RCGUI never starts rclone itself. This plugin is the exception that keeps that promise honest: it declares `proc` and `net` instead of `rclone`, so it spawns `rclone rcd` as a child process and health-checks the port on its own, then tells the host "connect here". Install, start, stop, autostart-on-launch and stop-on-exit all lived here before 0.4.1; that flow now lives in the app core, and the code stays in this repository as a reference.

Before 0.4.1, `config.json` wired the lifecycle to panel actions (`on_start` ran `auto_start`, `on_exit` ran `stop_on_exit`), so quitting RCGUI took the rclone child down with it. Both hooks are gone: the lifecycle belongs to the app core, and a plugin that manages it too would put two rclone processes on one port and have them kill each other on exit.

## What it used to do (before 0.4.1)

None of this runs now — it is listed so the migration reference and the code make sense. The app core performs every stage below itself.

| Stage | What it does |
| --- | --- |
| Find | Browse… for a specific file, Search this PC for a quick sweep of the usual locations, Deep scan for a full drive scan — only when you ask |
| Verify | Five pieces of evidence before a path counts as rclone: the file exists, it has a PE header, it carries a signature record, `rclone version` answers, and no earlier record contradicts it |
| Install | Download and install pulls the official build (or the enhanced build) for this machine's architecture and unpacks it next to the plugin's data |
| Start | `rcd --rc-addr 127.0.0.1:<port>` plus either `--rc-no-auth` or `--rc-user` / `--rc-pass`, as a silent child process with a log file |
| Connect | With Connect automatically on, the host is told to connect to that endpoint the moment rclone is up |
| Keep alive | Start with the app starts rclone whenever RCGUI starts; Stop rclone on exit decides whether it dies with the host |

## Install

- **Ships with the app.** RCGUI compiles `builtin` plugins into its own assets; they are absent from `plugins/manifest.json` and are never downloaded. Nothing to install by hand.
- **Only the Windows build ever installs a binary.** `config.json` says `"platform": "windows"`, so on other platforms the panel does not appear at all.
- **Uninstalling.** Disable it in RCGUI and the panel disappears; uninstalling deletes the package together with its settings. rclone copies already downloaded stay in the data directory — this plugin does not delete files for you.

## Panel

Four groups, top to bottom in the order the flow used to be worked through: pin down the executable, install if needed, settle the launch arguments, decide the start policy. Only the reading actions still do anything — `state`, browse, search, re-check and history. Download and install, start, autostart and stop answer with a deprecation notice and touch no process; the app core owns that job now.

Everything sits on one page, `settings` (`设置` / Settings), so a host that draws page tabs shows this panel as a single named group. The run-state line stays above the pages because the host renders one status line for the whole plugin. A host older than 0.3.0 ignores the page and lays every control out flat.

Labels below are the English strings; the panel carries the Chinese original next to every one of them (see [Languages](#languages)).

| Key | Control | Label | What it does |
| --- | --- | --- | --- |
| `conn` | status | Run state | Lamp: is rclone running, did verification pass |
| `g_exe` | group | ① rclone binary | Section one: the executable |
| `exe` | field | Program path | Type it or browse; the ▾ on the right is the history |
| `pick_exe` | action | Browse… | Host file dialog, `.exe` only |
| `search_exe` | action | Search this PC | Quick sweep of the usual install locations |
| `search_deep` | action | Deep scan | Walks the whole drive looking for `rclone.exe` |
| `revalidate` | action | Re-check | Runs the five checks again |
| `exe_history_hint` | hint | — | Explains the ▾: verified paths are recorded automatically, up to 10 |
| `checks` | checks | — | Verification state; reminds you to verify when it was never run |
| `g_install` | group | ② One-click install | Section two: download and install |
| `edition` | field | Edition | Official (default) or Enhanced |
| `source` | field | Download source | Official (default) or Mirrors first |
| `install_edition_hint` | hint | — | Reminds you the enhanced build only exists on GitHub Releases |
| `install` | action | Download and install | Detects the architecture, downloads the zip, unpacks it |
| `progress` | progress | — | Download progress |
| `install_hint` | hint | — | Per-stage install state |
| `g_args` | group | ③ Launch arguments | Section three: what `rcd` is started with |
| `rc_user` | field | User name | RC user name, `gui` by default |
| `rc_pass` | field | Password | May be left empty: a non-empty user name is enough to enable auth |
| `port` | field | Port | RC port, `5572` by default |
| `no_auth` | field | No auth locally | Adds `--rc-no-auth`; tick it only when auth truly is not needed |
| `silent` | field | Start silently | On by default: the child process gets no console window |
| `auto_connect` | field | Connect automatically | Tells the host to connect to this endpoint once rclone is up |
| `auth_warning` | hint | — | Appears here when the argument combination is wrong |
| `g_boot` | group | ④ Start policy | Section four: start with the app, die with the app |
| `auto_start` | field | Start with the app | Also the name of the `on_start` action |
| `stop_on_exit` | field | Stop rclone on exit | On by default: the child goes down with the host |
| `auto_start_hint` | hint | — | Speaks up when auto-start failed |
| `d_run` | divider | — | Separates the state from the buttons |
| `start` | action | Start rclone | Primary action: verify, launch, connect |
| `stop` | action | Stop | Stops rclone and clears the run state |

## What happens when you press Start

1. Five checks. Anything missing stops the launch here and names the gap under `checks` — starting an exe of unknown origin is far worse than reporting an error.
2. Pointless instances are refused. With No auth locally unticked and an empty user name, the plugin declines to start at all: such an instance answers 403 on everything except `core/version` and `rc/list`, which is useless to a host.
3. Arguments are assembled. `rcd --rc-addr 127.0.0.1:<port>`, then one of two branches: `--rc-no-auth`, or `--rc-user` plus `--rc-pass` (an empty password drops that flag, and rclone checks "user:empty" instead).
4. The host takes over. `proc.start` spawns the process with `silent` and `keep_on_exit`, and the log lands in `logs/oneclick-rclone.log` under the data directory.
5. Wait for the port, then connect. Nothing is reported as success until the port answers; with Connect automatically on, the host is connected right after.
6. Auto-start walks the same path, except a failure stays quiet — nobody wants an error popup at login, and the state lamp says what happened.

## Download sources

- Official: `version.txt` and the zip archives on `downloads.rclone.org`.
- Mirrors first: the Tsinghua and Nanjing University mirrors are tried before the official site; a mirror without the matching directory is skipped automatically.
- Enhanced: teldrive-backend builds appear only in the GitHub Releases of `gulp79/rclone-extra`, so mirrors are useless for them — this is the only restriction the Edition and Download source dropdowns impose on each other.
- The architecture comes from the host (`env.info`): x64, arm64 and 386 map to rclone's package names; when it cannot be worked out the plugin errors out instead of forcing a mismatched binary.

## Languages

The panel and every runtime message follow the host's UI language, and the strings live in the plugin rather than in the host.

- `config.json` carries each `label`, `placeholder` and hint `text` as `{"zh": …, "en": …}`, and `plugin_setting.version` is `2` so the host knows to resolve both. Dropdown options map a stable `value` (`官方版`, `镜像优先`, `模板一`) to a `label` in the same two languages — the value is what the plugin compares against, so the wire format never changes with the UI.
- Runtime messages (`status`, `toast`, `hints`, `checks`, manifest) go through `src/i18n.rs` and the `src/i18n.json` table compiled into the wasm. The message id is the Chinese original itself, so the source reads the way the string does, and placeholders keep the shape `{}` or `{name}`.
- The host passes `params.lang` on every call. Anything starting with `zh` — or no `lang` at all, as older hosts send — shows the Chinese original; everything else shows English, which means a new language costs nothing and degrades to English rather than to a blank panel.
- An id missing from the table falls back to the Chinese original, on purpose: a missed translation should leak a Chinese sentence, never an empty screen.

## Building

```powershell
rustup target add wasm32-wasip1
cargo build --release --target wasm32-wasip1
pwsh -File scripts/pack.ps1
```

Under bash, `bash scripts/pack.sh` does the same thing: build the wasm, assemble `dist/<id>-<version>/` (`config.json`, `main.wasm`, `check.sha256`), zip it, and name everything after the `id` this plugin actually uses (`onerclone`). Because it is a builtin plugin, a release needs no change to the host manifest — rebuilding RCGUI picks up the new version.

```
config.json                     identity, capabilities, the whole panel (no lifecycle hooks since 0.4.1)
src/main.rs                     the plugin itself, single file
src/i18n.rs                     language tag handling and message formatting
src/i18n.json                   Chinese message id -> English string
Cargo.toml                      wasm build settings and dependencies
scripts/pack.ps1                build, assemble dist/<id>-<version>, print sha256
scripts/pack.sh                 same, for bash and CI
.github/workflows/release.yml   manual workflow_dispatch -> build -> upload artifact (no Release)
.github/workflows/ci.yml        every push / PR: fmt --check, test, check(wasm32-wasip1)
MIGRATION.md                    moving the old kv data into the app core
CHANGELOG.md                    version history
```

To repackage, run the `release` workflow by hand from the Actions tab. It only uploads an artifact: it does not touch Releases and does not feed the host repository's plugin bundle.

## Plugin protocol

- The host runs `main.wasm` as a WASI command and feeds one JSON request per round on stdin.
- A round answers either with a result or with `{"need":[...]}`, asking the host for a capability call; the round continues once the host replies.
- Declared capabilities: `kv`, `fs`, `http`, `proc`, `net`, `env` — no `ui` (the panel lives entirely in `config.json`) and no `rclone` (this plugin talks to rclone itself).
- Install, verify and start are state machines: `step` is kept in `state` across rounds, so downloads and extraction can take as many rounds as they need.
- An unrecognised action answers `Unknown action: <name>` instead of breaking the round.

## Known limitations

- Windows only: a real `.exe` on disk, signature records and download sources beyond winget are Windows concerns.
- Deep scan walks an entire drive and is slow — which is why it only runs when the user clicks it.
- The enhanced build depends on a third-party repository (`gulp79/rclone-extra`); when upstream stops publishing builds, only older versions can be installed.
- Downloaded rclone copies are stored per version directory, and the plugin never cleans old versions up for you.
- The signature check only records that a signature exists; it does not verify that the signer is the rclone project.

## License

MIT — see [LICENSE](LICENSE).

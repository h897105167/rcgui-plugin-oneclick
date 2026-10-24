# rcgui-plugin-oneclick

**RCGUI 的「一键 rclone」插件** —— 搜索 / 校验 / 下载 / 安装本机 rclone，按面板上的启动参数拉起 `rclone rcd`，并可随程序自动启动、退出时停止。

> 本体仓库：[h897105167/rclone_gui_windows](https://github.com/h897105167/rclone_gui_windows)
> 插件开发文档：[docs/PLUGIN.md](https://github.com/h897105167/rclone_gui_windows/blob/main/docs/PLUGIN.md)

本仓库**只包含插件自己**：Rust 源码、`config.json`、打包脚本、发版 CI。
面板不是代码，而是 `config.json` 里的 `plugin_setting` 声明 —— 本体按声明渲染，不认识「一键 rclone」这个名字。

> 注意：本体不拥有 rclone 的进程生命周期，它不启动/停止/重启 rclone。
> 「启动 rclone」是**这个插件**主动做的事，用的是插件自己的 `proc` 能力，与本体无关。

---

## 安装

### 内置分发（推荐）

内置分发：构建本体时 `scripts/fetch_plugins.ps1` 按 `plugins/manifest.json` 登记的版本与本仓库 [Release](https://github.com/h897105167/rcgui-plugin-oneclick/releases) 下载 zip、校验 sha256，解包到 `crates/app/assets/plugins/onerclone/`；本体启动时再释放到 `%APPDATA%\RCGUI\plugins\onerclone\`。
升级 = 改本体 manifest 里的 `tag` 与 `sha256`（zip 的哈希）。

### 手动

1. 从 [Releases](https://github.com/h897105167/rcgui-plugin-oneclick/releases) 下载 `onerclone-<版本>.zip`；
2. 解压到 `%APPDATA%\RCGUI\plugins\onerclone\`，目录里应有 `config.json`、`main.wasm`、`check.sha256`；
3. 重启 RCGUI，在「插件」页看到「一键 rclone」即可。

---

## 功能

| 分组 | 说明 |
|---|---|
| ① rclone 程序 | 选 `rclone.exe`、搜本机（常规 / 深度扫描）、重新校验；通过校验的路径自动记入历史（最多 10 条） |
| ② 一键安装 | 官方版 / 增强版（teldrive 后端），官方源 / 镜像优先；下载、解压、校验、落盘，带进度 |
| ③ 启动参数 | 用户名 / 密码 / 端口 / 本地不校验 / 静默启动 / 自动连接 |
| ④ 启动策略 | 随程序自动启动、退出时停止 |
| 运行 | 启动 rclone（`rcd`）、停止 |

校验不只看文件名：会跑 `rclone version` 确认是可执行的 rclone，并解析出版本号。

## 设置项

| 控件 key | 类型 | 说明 |
|---|---|---|
| `conn` | status | 运行状态（带指示灯） |
| `exe` | path | rclone 程序路径 |
| `pick_exe` / `search_exe` / `search_deep` / `revalidate` | action | 浏览、搜索本机、深度扫描、重新校验 |
| `edition` / `source` | select | 版本（官方版/增强版）、下载源（官方源/镜像优先） |
| `install` | action | 一键下载并安装 |
| `progress` | progress | 下载/解压进度 |
| `rc_user` / `rc_pass` / `port` / `no_auth` / `silent` / `auto_connect` | 文本/密码/数字/开关 | 启动参数 |
| `auto_start` / `stop_on_exit` | bool | 启动策略（同时也作为 `on_start` / `on_exit` 的开关） |
| `start` / `stop` | action | 启动 / 停止 rclone |

---

## 开发

需要 Rust 1.80+ 和 `wasm32-wasip1` 目标：

```powershell
rustup target add wasm32-wasip1
cargo build --release --target wasm32-wasip1   # 产物 target/wasm32-wasip1/release/main.wasm
```

打包（生成目录包 + `check.sha256` + zip，并打印 zip 的 sha256）：

```powershell
pwsh -File scripts/pack.ps1        # Windows
bash scripts/pack.sh               # Linux / macOS / CI
```

### 目录结构

```
.
├── Cargo.toml              # package oneclick-plugin, [[bin]] name = "main"
├── config.json             # 插件清单 + 声明式面板（本体只读这个）
├── src/
│   ├── main.rs             # 入口：stdin 读一个 JSON，stdout 写一个 JSON
│   └── ...
├── scripts/
│   ├── pack.ps1            # Windows 打包
│   └── pack.sh             # CI / *nix 打包
└── .github/workflows/
    └── release.yml         # 打 tag → 构建 → 打包 → 发 Release
```

### 发版

1. 改 `config.json` 的 `version`（必须与 tag 一致，CI 会校验）；
2. `git tag v0.3.1 && git push origin v0.3.1`；
3. CI 构建、打包、建 Release，资产名 `onerclone-0.3.1.zip`；
4. 把 Release 页面上打印的 sha256 填进本体的 `plugins/manifest.json`。

---

## 协议

插件是一个 **WASI 命令行程序**：每次调用，本体把一段 JSON 写到 stdin，插件把一段 JSON 写到 stdout。
需要宿主能力时，插件返回 `{"need":[{"cap":"...","params":{...}}]}`，本体执行后带着结果重新运行插件。
完整协议、能力清单、面板 schema、命名规则、接入与分发规则见本体的
**[docs/PLUGIN.md](https://github.com/h897105167/rclone_gui_windows/blob/main/docs/PLUGIN.md)**。

本插件声明的能力：`kv`（设置与历史）、`fs`（查找/解压/写文件）、`http`（下载 rclone 压缩包）、
`proc`（启动与探测 rclone 进程）、`net`（端口探测）、`env`（数据目录、系统信息）。

---

## 已知限制

* 目前只支持 Windows（`config.json` 的 `platform` 是 `windows`）；其他平台本体不会展示它。
* 增强版（teldrive 后端）只发布在 GitHub Releases，镜像源对它不可用。
* 插件负责启动 rclone，但**不接管**用户自己启动的 rclone；「停止」只停插件启动的那个进程。

# rcgui-plugin-oneclick

**RCGUI 的一键 rclone 插件** —— 找到本机的 `rclone.exe`、确认它真的是 rclone、没有就下一个、以 `rclone rcd` 拉起来，再把 RC 端点交回宿主。

[English](README.md) | 简体中文

> 本体仓库：[h897105167/rclone_gui_windows](https://github.com/h897105167/rclone_gui_windows)
> 插件开发文档：[docs/PLUGIN.md](https://github.com/h897105167/rclone_gui_windows/blob/main/docs/PLUGIN.md)

本仓库只放插件自己：Rust 源码、`config.json`、打包脚本、发版 CI。你在面板里看到的每个控件都由 `config.json` 声明，表单是宿主画出来的。

这个插件是 `"builtin": true`、`"platform": "windows"`：它随 RCGUI 的资源一起分发，从不经过 `plugins/manifest.json` 下载。它也是唯一被允许对 Windows 作假设的插件，因为这条流程里 rclone 的官方构建就是 Windows 二进制。

RCGUI 自己从不启动 rclone。这个插件让那句承诺依然成立：它声明的是 `proc` 与 `net`，不是 `rclone` —— 由它把 `rclone rcd` 当子进程拉起来、自己探端口，然后告诉宿主"连这里"。安装、启动、停止、随程序启动、退出时停止，全都住在这里。

`config.json` 把生命周期挂在面板动作上：`on_start` 跑 `auto_start`，`on_exit` 跑 `stop_on_exit` —— 所以退出 RCGUI 会带走那个 rclone 子进程，除非你把这一项关掉。

## 功能

| 环节 | 做什么 |
| --- | --- |
| 找 | `浏览…` 指定文件、`搜索本机` 快扫常见位置、`深度扫描` 翻遍磁盘 —— 只在你要的时候 |
| 验 | 一个路径要算作 rclone，得凑齐五项证据：文件存在、有 PE 头、带签名记录、`rclone version` 有应答、旧记录不矛盾 |
| 装 | `一键下载并安装` 按本机架构取官方构建（或增强版），解压到插件数据目录旁边 |
| 启 | `rcd --rc-addr 127.0.0.1:<端口>`，再加 `--rc-no-auth` 或 `--rc-user` / `--rc-pass`，作为静默子进程跑，带日志文件 |
| 连 | 勾着 `自动连接` 时，rclone 一起来就让宿主连上这个端点 |
| 守 | `随程序自动启动` 让 rclone 随 RCGUI 起来；`退出时停止 rclone` 决定它是否随宿主一起走 |

## 安装

- **随本体分发。** RCGUI 把 `builtin` 插件编进自己的资源，`plugins/manifest.json` 里没有它，也不会去下载它。不需要手动安装。
- **只有 Windows 构建会被安装。** `config.json` 写着 `"platform": "windows"`，别的平台干脆不出现这个面板。
- **卸载。** 在 RCGUI 里禁用，面板就消失；卸载会连包带设置一起删。已下载的 rclone 副本留在数据目录里 —— 这个插件不会替你删文件。

## 面板

四个分组，从上到下就是我建议的顺序：先确定可执行文件，再装，再定参数，最后定启动策略。

| 键 | 控件 | 标签 | 作用 |
| --- | --- | --- | --- |
| `conn` | status | 运行状态 | 指示灯：rclone 在不在跑、校验过没过 |
| `g_exe` | group | ① rclone 程序 | 第一节：可执行文件 |
| `exe` | path | 程序路径 | 手填或浏览；右侧 ▾ 是历史记录 |
| `pick_exe` | action | 浏览… | 宿主文件对话框，只挑 `.exe` |
| `search_exe` | action | 搜索本机 | 常见安装位置快扫 |
| `search_deep` | action | 深度扫描 | 翻遍磁盘找 `rclone.exe` |
| `revalidate` | action | 重新校验 | 重跑一次五项校验 |
| `exe_history_hint` | hint | — | 说明右侧 ▾：通过校验的路径自动记入，最多 10 条 |
| `checks` | checks | — | 校验状态；没校验过时会提醒你先校验 |
| `g_install` | group | ② 一键安装 | 第二节：下载与安装 |
| `edition` | select | 版本 | `官方版`（默认）或 `增强版` |
| `source` | select | 下载源 | `官方源`（默认）或 `镜像优先` |
| `install_edition_hint` | hint | — | 提醒增强版只在 GitHub Releases 发布 |
| `install` | action | 一键下载并安装 | 探测架构、下载 zip、解压 |
| `progress` | progress | — | 下载进度 |
| `install_hint` | hint | — | 逐阶段的安装状态 |
| `g_args` | group | ③ 启动参数 | 第三节：`rcd` 的启动参数 |
| `rc_user` | text | 用户名 | RC 用户名，默认 `gui` |
| `rc_pass` | password | 密码 | 可以留空：用户名非空就够启用鉴权了 |
| `port` | number | 端口 | RC 端口，默认 `5572` |
| `no_auth` | bool | 本地不校验 | 开启 `--rc-no-auth`，只在确实不需要鉴权时才勾 |
| `silent` | bool | 静默启动 | 默认勾上：不给这个子进程开控制台窗口 |
| `auto_connect` | bool | 自动连接 | 启动完成后让宿主连上这个端点 |
| `auth_warning` | hint | — | 参数组合有问题时出现在这里 |
| `g_boot` | group | ④ 启动策略 | 第四节：随程序起来，随程序退出 |
| `auto_start` | bool | 随程序自动启动 | 也是 `on_start` 的动作名 |
| `stop_on_exit` | bool | 退出时停止 rclone | 默认勾上：宿主退出时把子进程一起关掉 |
| `auto_start_hint` | hint | — | 自动启动失败时给出提示 |
| `d_run` | divider | — | 把状态与按钮隔开 |
| `start` | action | 启动 rclone | 主操作：校验、拉起、连接 |
| `stop` | action | 停止 | 停掉 rclone 并清掉运行状态 |

## 启动时到底发生了什么

1. 五项校验。没通过就停在这里，并把缺口写在 `校验` 里 —— 拉起一个来历不明的 exe 比报错糟得多。
2. 拒绝无意义的实例。没勾 `本地不校验`、用户名又空着时，插件直接拒绝启动：那样拉起的实例只会对 `core/version`、`rc/list` 之外的所有接口返回 403，对宿主毫无用处。
3. 拼参数。`rcd --rc-addr 127.0.0.1:<端口>`，然后二选一：`--rc-no-auth`，或者 `--rc-user` 加 `--rc-pass`（密码为空就不带这个参数，rclone 会按"用户:空密码"校验）。
4. 交给宿主。`proc.start` 起进程，带上 `silent` 与 `keep_on_exit`，日志写到数据目录下的 `logs/oneclick-rclone.log`。
5. 等端口、再连接。端口通之前不报成功；`自动连接` 开着的话，接着让宿主连上去。
6. 自动启动走同一条路，只是失败时不弹窗 —— 开机时没人想看一条报错，状态灯会说明一切。

## 下载源

- `官方源`：`downloads.rclone.org` 上的 `version.txt` 与 zip 包。
- `镜像优先`：先试清华与南大镜像，再退回官方源；某个镜像没有对应目录时会自动跳过。
- `增强版`：teldrive 后端的构建只出现在 `gulp79/rclone-extra` 的 GitHub Releases 里，镜像源对它无效 —— 这是 `版本` 与 `下载源` 两个下拉唯一的组合限制。
- 架构取自宿主（`env.info`）：x64、arm64、386 各自对应 rclone 的包名，猜不出来就直接报错，不硬塞一个不匹配的二进制。

## 构建

```powershell
rustup target add wasm32-wasip1
cargo build --release --target wasm32-wasip1
pwsh -File scripts/pack.ps1
```

bash 下用 `bash scripts/pack.sh`，两个脚本等价：构建 wasm、组装 `dist/<id>-<版本>/`（`config.json`、`main.wasm`、`check.sha256`）、打成 zip 并按本插件实际使用的 `id`（`onerclone`）命名。由于它是内置插件，发版不需要改宿主的 manifest —— 重新构建 RCGUI 就会带上新版本。

```
config.json                     身份、能力、生命周期与整个面板
src/main.rs                     插件本体，单文件
Cargo.toml                      wasm 构建参数与依赖
scripts/pack.ps1                构建、组装 dist/<id>-<版本>、打印 sha256
scripts/pack.sh                 同上，给 bash 与 CI 用
.github/workflows/release.yml   tag -> 校验版本 -> 构建 -> GitHub Release
```

## 插件协议

- 宿主把 `main.wasm` 当 WASI 命令跑，每轮从 stdin 读一条 JSON 请求。
- 一轮的应答要么是结果，要么是 `{"need":[...]}`，向宿主申请能力调用；宿主答复后这一轮继续。
- 声明能力 `kv`、`fs`、`http`、`proc`、`net`、`env` —— 没有 `ui`（面板全靠 `config.json`），也没有 `rclone`（这个插件自己和 rclone 打交道）。
- 安装、校验、启动都是状态机：`step` 存在 `state` 里过轮次，所以下载和解压可以跨很多轮慢慢走。
- 遇到不认识的 action 会回 `未知动作: <名字>`，而不是把这一轮弄崩。

## 已知限制

- 只支持 Windows：真机上的 `.exe`、签名记录、winget 之外的下载源都是 Windows 专属。
- `深度扫描` 会遍历整块磁盘，很慢 —— 所以它只在用户点它时才跑。
- 增强版依赖第三方仓库（`gulp79/rclone-extra`），上游不发新构建时只能装到旧版本。
- 已下载的 rclone 副本按版本号目录存放，插件不会替你清理旧版本。
- 数字签名检查只记录"签名存在"，不验证签发者是不是 rclone 官方。

## 许可证

MIT —— 见 [LICENSE](LICENSE)。

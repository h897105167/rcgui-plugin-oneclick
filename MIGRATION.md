# 从「一键 rclone」插件迁到主程序内核

插件 `onerclone`（一键 rclone）的安装 / 启动 / 停止能力已经并进主程序。这个仓库
从此只留源码作参考，插件不再发行新版本，也不再随 tag 发 Release。

这份文档写给两种人：手上还有旧插件数据、想搬到主程序里的用户；以及要维护主程序
内核那段 rclone 逻辑的开发者。第 3 节是给后者的。

## 1. 旧插件在 kv 里存了哪些键

旧插件的所有持久化数据都在宿主给它的 kv 里（plugin id 为 `onerclone`）。分两类：
**用户设置**（面板上填的那些）和 **`_state` 一个内部快照**。

### 1.1 用户设置（宿主按 `config.json` 的 controls 自动存取）

| key | 类型 | 示例值 | 含义 |
| --- | --- | --- | --- |
| `exe` | string | `C:\Users\me\AppData\Local\RCGUI\rclone\v1.75.2\rclone.exe` | 选定的 rclone.exe 绝对路径 |
| `edition` | string | `官方版` / `增强版` | 下载哪条线：官方 rclone，还是带 teldrive 后端的增强版 |
| `source` | string | `官方源` / `镜像源` | 官方版的下载源优先级；值以「镜像」开头就先试镜像 |
| `rc_user` | string | `gui` | `--rc-user`；留空且没勾 no_auth 时插件拒绝启动（见第 3.4 节） |
| `rc_pass` | string | `1` | `--rc-pass`；可以留空，rclone 允许只给用户名 |
| `port` | string | `5572` | rclone RC 的监听端口，默认 5572 |
| `no_auth` | bool | `false` | 勾了就加 `--rc-no-auth`，本地不校验用户名密码 |
| `silent` | bool | `true` | 拉起的进程是否隐藏窗口 |
| `auto_connect` | bool | `true` | 启动成功后是否顺手把 RC 端点交给宿主去连 |
| `auto_start` | bool | `false` | 宿主启动时是否自动拉起 rclone（生命周期动作，已停用） |
| `stop_on_exit` | bool | `true` | 宿主退出时是否停掉这个 rclone（生命周期动作，已停用） |

`edition` 和 `source` 存的是**中文标签原文**，不是枚举值，判断时用 `starts_with("增强")`
/ `starts_with("镜像")`。迁移时不要把它当规范值转义，照抄即可。

### 1.2 `_state`：内部快照（插件自己写，宿主不该读）

`_state` 是一个 JSON 对象，整体一个 kv 键：

```json
{
  "proc": { "handle": "proc-7f3a", "pid": 23184 },
  "checks": [
    { "label": "文件存在", "value": "C:\\...\\rclone.exe", "ok": true },
    { "label": "体积", "value": "62.4 MB", "ok": true }
  ],
  "exe_version": "v1.75.2",
  "exe_history": [
    "C:\\Users\\me\\AppData\\Local\\RCGUI\\rclone\\v1.75.2\\rclone.exe",
    "D:\\tools\\rclone.exe"
  ],
  "verified_exe": "C:\\Users\\me\\AppData\\Local\\RCGUI\\rclone\\v1.75.2\\rclone.exe",
  "search_deep_offered": true
}
```

| 字段 | 类型 | 含义 | 迁移时 |
| --- | --- | --- | --- |
| `proc` | object `{handle, pid}` 或空 | 由本插件启动的那个进程的句柄与 PID | **丢掉**，句柄只在本进程树里有意义 |
| `checks` | array of `{label, value, ok}` | 上一次校验的六项清单（文件存在、文件名、体积、程序格式、数字签名、可执行） | **丢掉**，是给界面看的旧结论 |
| `exe_version` | string | 上次校验通过的版本，如 `v1.75.2` | **丢掉**，重新跑一次 `rclone version` 就有了 |
| `exe_history` | array of string | 用过的 exe 路径，最新的在前，最多 10 条 | **保留**，这是用户手挑过的路径，丢了要重新找 |
| `verified_exe` | string | 上次校验通过的那个路径 | **丢掉**，缓存而已；文件和当初不是同一份也不会变 |
| `search_deep_offered` | bool | 是否已经提示过「要不要深度扫描」 | 随便，丢掉只是会再提示一次 |

要点：`verified_exe` 记的是**路径**而不是文件指纹，所以它永远不能当成「这个 exe 仍然
可信」的凭据 —— 换掉同名文件它照样通过。校验结论一律重算。

## 2. 迁移算法

### 2.1 读取顺序

1. 读插件 kv 的 `exe`、`port`、`rc_user`、`rc_pass`、`no_auth`、`silent`、
   `auto_connect`；有值就写进主程序同名字段。
2. 读 `_state.exe_history`，整份复制成主程序的「最近用过的 rclone」列表。
3. 读 `_state.exe_version`，**只**用作界面提示（「旧插件记的是 v1.75.2」），
   不当作已校验。
4. 其它 `_state` 字段全部忽略。

### 2.2 必须保留的键

- `port`：用户可能特意避开被占用的端口，重算成默认 5572 会让别人的脚本连不上。
- `rc_user` / `rc_pass` / `no_auth`：改掉任何一个，之前存进主程序的 RC 连接参数就全废了。
- `exe`：用户手选的路径，可能在自己整理的目录里，不在 `%LOCALAPPDATA%\RCGUI\rclone`。
- `_state.exe_history`：同上，而且它是重新找回 exe 的唯一线索。
- `edition` / `source`：影响下次装哪条线、走不走镜像。

### 2.3 可以丢弃的键（重算比继承更可靠）

- `_state.verified_exe`、`_state.exe_version`、`_state.checks`：校验缓存。必须重跑一遍
  `<exe> version`，否则主程序会信任一个可能已经被替换掉的文件。
- `_state.proc`：旧句柄。旧插件已经不可能在运行了（宿主不会再给它 `proc.start`），
  留着它只会让「要不要停掉自己启动的进程」的逻辑判断错。
- `_state.search_deep_offered`：纯 UI 状态。
- `auto_start`、`stop_on_exit`：**这两项直接作废，不要迁移**。它们的作用是让插件和
  宿主抢着管 rclone 进程，这正是弃用的原因。主程序有自己的开机启动开关，让用户去那
  里设。

### 2.4 迁完之后

旧 kv 不要删：`MIGRATION.md` 的存在意义之一就是「数据一条没删」。旧插件已不会再被
调度（`config.json` 里 `on_start` / `on_exit` 已移除，源码里的守门也会拒掉安装 /
启动 / 停止），留着它不构成双管内核。

## 3. 主程序内核要复用的部分

下面这些是旧插件的实际行为，`src/main.rs` 里原样留着（`DEPRECATED = false` 时才跑得到）。
内核实现要么照抄，要么能解释为什么不一样。

### 3.1 版本解析：两条线

**官方版**（`resolve_official`，src/main.rs:1584）

1. 依次对候选源请求 `<base>/version.txt`。候选源顺序由 `source` 决定：
   先镜像（`https://mirrors.tuna.tsinghua.edu.cn/rclone`、`https://mirrors.nju.edu.cn/rclone`），
   再官方 `https://downloads.rclone.org`。
2. 从 `version.txt` 首行抽出版本号：去掉行首 `rclone`，取第一个词，必须是 `v` 加数字
   （`parse_version_txt`，src/main.rs:1569）。抽不出就换下一个源。
3. 拼下载地址：`{base}/{version}/rclone-{version}-windows-{arch}.zip`。
4. 架构映射：`x86_64 → amd64`、`x86 → 386`、`aarch64 → arm64`，其它直接报错。

**增强版**（`resolve_enhanced`，src/main.rs:1621）

1. 请求 `https://api.github.com/repos/gulp79/rclone-extra/releases/latest`。
2. 必须有 `tag_name`，tag 即版本号。
3. 在 `assets` 里找名字等于 `rclone-windows-{arch}.zip`（不区分大小写）的资产，用它的
   `browser_download_url`；没有 `assets` 字段时退回固定地址
   `https://github.com/gulp79/rclone-extra/releases/download/{tag}/rclone-windows-{arch}.zip`。
4. 增强版只发在 GitHub，镜像源帮不上忙，所以这条路不走镜像。

下载体积上限 400 MB，解压后上限 600 MB（`MAX_DOWNLOAD` / `MAX_EXTRACTED`）。

### 3.2 exe 校验：六项，最后一项才是硬证据

`judge`（src/main.rs:1741）的判定链，一条比一条硬：

| 项 | 怎么判 | 是否参与最终结论 |
| --- | --- | --- |
| 文件存在 | `fs.meta` 的 `exists && is_file` | 是 |
| 文件名 | 文件名（小写）含 `rclone` | 否，只作软信号 |
| 体积 | 4 MB ≤ size ≤ 300 MB | 是 |
| 程序格式 | `fs.pe_info.ok`，真 PE 文件 | 是 |
| 数字签名 | `fs.signature`：`valid` 且签名者含 `rclone` 才算过；`NotSigned` 放行 | 是 |
| 可执行 | 跑 `<exe> version`，超时 20 秒，退出码 0 且 stdout 里有 `rclone v...` | 是 |

`NotSigned` 放行是故意的：官方 rclone 的 Windows 二进制本来就没有正式签名，拦掉它
等于拦掉所有人。反过来，签名有效但签名者不是 rclone，说明系统里有同名的别的程序，
必须拦。

### 3.3 安装路径

`install_roots`（src/main.rs:1082）：

- 宿主提权时优先装进 `%ProgramFiles%\RCGUI\rclone\{version}\rclone.exe`；
- 否则装到 `%LOCALAPPDATA%\RCGUI\rclone\{version}\rclone.exe`；
- 系统目录写不进时回落到用户目录，并在界面上说一句。

`{version}` 会先过 `safe_dir_name`（src/main.rs:1699），非字母数字点横线一律换成 `_`，
防止版本串里夹路径分隔符。

### 3.4 启动参数拼装与 RC 认证

```
rclone rcd --rc-addr 127.0.0.1:{port} [--rc-no-auth | --rc-user {user} [--rc-pass {pass}]]
```

- `no_auth` 为真 → 加 `--rc-no-auth`；否则分别加 `--rc-user` / `--rc-pass`，密码可以为空。
- **用户名留空且没勾 no_auth 时不要在自动启动里静默拉起**。原因是 rclone 只放行注册
  为 NoAuth 的接口（`core/version`、`rc/list` 等），其余 `options/get`、`config/paths`、
  `mount/mount`、`serve/start` 全部 403 —— 界面上表现成「连上了，但 5 项自动恢复全失败」。
  旧插件在这种情况下直接拒绝启动（用户手动点时给红色错误，自动启动时只给一行警告）。
- 进程参数：`silent` 决定是否隐藏窗口；`keep_on_exit = !stop_on_exit`；日志写到
  `{data_dir}/logs/oneclick-rclone.log`。内核要自己决定日志放哪，但请保留「谁启动谁
  负责停」这条规矩，别和插件并存。

## 4. 用户怎么迁（照着做三分钟）

1. **先关掉插件。** 在插件的「一键 rclone」面板上，如果「运行状态」显示正在运行，
   先点一次「停止（已弃用）」（这个动作现在只会回一句提示；如果进程真的还在，用任务
   管理器结束那个 `rclone.exe`）。确认没有插件在管 rclone 了再往下走。
2. **把同样的参数填进主程序。** 端口、用户名、密码、是否免校验、exe 路径，照旧插件的
   填法抄过去。旧插件里的值不会自动搬过去 —— 去看一眼再填，比事后猜为什么连不上快。
3. **确认没有两个 rclone 进程。** 打开 PowerShell 跑：

   ```powershell
   Get-Process rclone -ErrorAction SilentlyContinue |
     Select-Object Id, StartTime, Path
   ```

   期望结果：只有一行（主程序启动的那个）。两行以上说明还有东西在拉 rclone，看
   `Path` 和 `StartTime` 分辨是谁；把多出来的那个结束掉，再去主程序里确认它自己的
   那个还活着。
4. **顺手确认端口没被抢。** `netstat -ano | findstr :5572`，同一个端口只能有一个
   监听者（最后一列是 PID）。两个进程抢同一个端口时，后起的那个会启动失败并把
   错误写进日志，界面上看起来像「连不上」。
5. 旧插件的设置和历史**不需要备份也不需要删除**，它们留在 kv 里；主程序不会读它们，
   也不会因为它们在而改变行为。

## 5. English summary

The `onerclone` plugin is deprecated: installing, starting and stopping rclone is the
app's job now. Its settings and history are still in the plugin kv store and were not
deleted.

To migrate: keep `port`, `rc_user`, `rc_pass`, `no_auth`, `silent`, `auto_connect`,
`exe`, `edition`, `source` and `_state.exe_history`; discard `auto_start`,
`stop_on_exit` and everything else under `_state` (`proc`, `checks`, `exe_version`,
`verified_exe`, `search_deep_offered`) because those are caches or handles that must be
recomputed. Then enter the same values in the app, and check with
`Get-Process rclone` that exactly one rclone process is running.

Section 3 describes the logic the app core took over: version resolution from
`downloads.rclone.org` and from the `gulp79/rclone-extra` GitHub release, the six-step
exe check (size 4–300 MB, PE header, signature, `rclone version` output), the
`rclone rcd --rc-addr 127.0.0.1:{port}` argument assembly, and why an empty RC user name
must not be launched silently.

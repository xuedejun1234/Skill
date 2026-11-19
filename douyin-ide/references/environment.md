# 环境、账号与项目窗口

## CLI 可用性

日常使用直接调用 `douyinide`。仅在命令不存在、无法执行或用户要求配置时检查当前安装位置：macOS/Linux 使用 `command -v douyinide`，Windows PowerShell 使用 `Get-Command douyinide`，`cmd.exe` 使用 `where douyinide`。命令不存在时通过 `bytedanceide://` 协议唤起 IDE（见 SKILL.md「唤起 IDE」），由工具注册 CLI。唤起后命令仍不存在，或协议无法启动并连接进程时，直接提示用户前往[抖音开发者工具下载页](https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/dev-tools/developer-instrument/download/developer-instrument-update-and-download)下载最新版本；不要猜测固定的应用安装路径，也不要把 npm 安装或修改系统链接作为业务操作的默认前置。

Windows 安装包会将 `douyinide.cmd` 命令 shim 安装到当前用户的 `WindowsApps` 目录，并将 skill 目录联接到 `%USERPROFILE%\.agents\skills\douyin-ide`。安装完成后新开的 PowerShell 或 `cmd.exe` 可直接调用 `douyinide`；已有终端若未识别命令，先重新打开终端。

命令需要访问本机 IDE 的服务与用户数据。若执行环境无法访问宿主机，说明具体连接或权限阻塞，改用获准的本机执行通道；不要通过手写 HTTP 请求绕过 CLI 授权。纯文档或本地代码修改不需要启动 IDE。

## 状态与登录

```bash
douyinide check-ide-status -c <clientName>
```

| 返回字段 | 含义与处理 |
|----------|------------|
| `ideOnline` | 是否连通 IDE 健康接口；为 false 时，需要窗口的任务可通过开窗命令启动 IDE |
| `loginExpired` | 登录状态；为 true 时，依赖账号的操作先执行登录流程 |
| `loginUser` | 有效登录时可能返回的账号信息 |
| `skillVersion` / `skillPath` | 当前 CLI 对应 skill 版本与目录；不是 agent/IDE 版本比较结果 |

健康检查会把连接异常归为离线，单凭 `ideOnline: false` 不能断定进程不存在。必要时用目标命令的实际错误进一步定位。当前版本没有 `--skill-version`、`versionRelation`、`--token`、`taskId` 或轮询命令。

```bash
douyinide login -c <clientName>
```

有效登录且 IDE 离线时会启动进程，再返回 `loggedIn: true`，但这不保证窗口服务已经就绪。未登录时会启动 IDE，返回 `ok: true, loggedIn: false`，由用户在 IDE 中完成登录。此时说明需完成登录，后续继续前重新检查状态；不要把 `ok: true` 当作已登录，也不要不断重发 `login`。

## 发现项目与 App ID

```bash
douyinide project-list -c <clientName>
douyinide get-user-appids -c <clientName>
douyinide get-project-windows-info -c <clientName>
```

- `project-list` 读取已导入列表，包含 `path`、`name`、`projectType`，不要求 IDE 在线。它不是打开窗口的列表，也不会导入或创建项目。
- `get-user-appids` 使用 IDE 登录信息查询当前账号有权限的小程序和小游戏，返回 `appids` 数组。无需开项目窗口，但需要有效登录及网络。不要从项目名称猜 App ID。
- `get-project-windows-info` 查询在线 IDE 的 `codeWindows`。按项目路径匹配目标，使用返回的真实 `id`；不要根据当前焦点猜窗口。多个候选无法由任务确定时再让用户明确项目。

需要继续编译、截图、日志或交互时，把匹配到的项目根、项目类型和真实窗口 ID 写入 [IDE 运行状态链](runtime-state-chain.md) 的 `runtimeContext`。同一次验证流程固定使用该 `windowId`，不要中途改回路径定位；窗口关闭、IDE 重启或连接重建后重新查询，不复用旧 ID。

## 导入和分享代码片段

```bash
douyinide import-minicode -c <clientName> --url <snippetUrl>
douyinide share-minicode -c <clientName> --path <projectPath>
douyinide share-minicode -c <clientName> --path <projectPath> --expire <timestampMs>
```

两个命令都需要 IDE 主进程；离线时自动启动并等待服务就绪，不打开项目窗口，沿用客户端授权。

- `import-minicode` 只需要 HTTP(S) 代码片段链接，使用 IDE 配置的默认项目目录和自动生成的 `minicode-N` 名称。成功返回 `targetDir`、`projectName`、`projectType`、`trusted: false`，并加入项目列表。导入后先检查代码安全性，再按需打开运行。
- `share-minicode` 从 `--path` 读取项目配置，复用 IDE 的打包和上传规则，返回分享 `url` 和 `expire`。分享需要有效登录；这是上传本地代码的操作，仅在用户要求分享时执行。
- `--expire` 为未来 365 天内的 Unix 毫秒时间戳；省略或传 `0` 表示不过期，不是持续秒数。
- 业务失败返回 `ok: false`，命令退出码为非零。结果不确定时先核实状态，不自动重复导入或上传。

## 打开与关闭

```bash
douyinide open-project-window -c <clientName> --path <projectPath> --type app [--mode full|lite]
```

该命令按本地绝对路径打开项目窗口；路径尚未在 IDE 项目列表中时会一并导入。`--path` 始终传入含 `project.config.json` 的项目根目录，不要把 `miniprogramRoot` 当作项目路径。`--type`：小程序用 `app`，小游戏用 `microgame`（含 Unity 小游戏，不要传 `unity`）。用户已给出类型时以用户为准；未给出时先读项目根的 `project.config.json`：若有 `miniprogramRoot`，在 `{项目根}/{miniprogramRoot}/app.json` 判断小程序，否则在项目根查找 `app.json`；`game.json` 始终在项目根判断小游戏。两者都有或都没有时再询问，不要从目录名或已导入列表猜测。`--mode` 仅支持 `full` 和 `lite`，省略时默认 `lite`；只有用户明确要求完整模式时才传 `full`，Unity 项目由 IDE 强制使用完整模式。进入 Lite 模式前，IDE 会先把该项目的 `liteToolbarDeviceTypeStatus` 设置为 `simulator`，确保开窗时初始化模拟器。命令发现 IDE 离线时自动启动，每 2 秒检查一次，最多等 60 秒；就绪后检查登录，未登录时需先登录。开窗完成后查询窗口列表，再进行模拟器操作。

所有窗口操作二选一指定 `--window-id <windowId>` 或 `--project-path <projectPath>`。路径无匹配表示目标未打开；多窗口匹配同一路径时使用查询确认的窗口 ID。

```bash
douyinide close-project-window -c <clientName> --window-id <windowId>
douyinide close-project-window -c <clientName> --project-path <projectPath>
```

以上是两种替代写法，不需要都运行。关闭命令仍兼容旧 `--path`，新调用统一使用窗口目标参数。

```bash
douyinide logout -c <clientName>
douyinide quit -c <clientName>
```

`logout` 注销当前账号：在线时走 IDE 完整注销逻辑，离线时清空本地登录信息。仅在用户需要注销/切换账号时执行。`quit` 关闭整个 IDE 主进程和所有项目窗口；关闭单项目用 `close-project-window`。不要作为任务结束后的自动清理。

## 失败处理

| 现象 | 处理 |
|------|------|
| 命令不存在或启动器执行失败 | 用 `bytedanceide://` 协议唤起一次以注册 CLI。唤起后命令仍不存在或协议失败，直接给出[最新版下载页](https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/dev-tools/developer-instrument/download/developer-instrument-update-and-download)，不再尝试启动，不枚举路径 |
| CLI 已存在，但提示工具未运行、找不到服务信息、连接拒绝 | 执行一次 `douyinide start-ide`；超时或启动失败时直接给出[最新版下载页](https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/dev-tools/developer-instrument/download/developer-instrument-update-and-download)，不改用协议重试 |
| `client_not_authorized` | 提示在 IDE 授权当前客户端；拒绝后停止，不改客户端名重试 |
| 登录失效、App ID 无权限 | 完成登录或确认目标账号/项目权限；不自行修改登录文件或输出 Cookie |
| 找不到窗口或同路径多个窗口 | 刷新窗口列表，定位真实 ID；未开窗时先打开 |
| 项目配置不存在、类型错误 | 根据报错核对项目根、`project.config.json` 的 `miniprogramRoot` 与对应位置的 `app.json`/`game.json`，不为通过检查伪造配置 |
| 模拟器未就绪、页面不存在、元素失效 | 确认加载状态和当前页，重新取快照后再操作 |
| QA 方法超时 | 服务端 QA 调用超时为 60 秒；报告失败步骤和已知状态，不无限重试 |
| `ok: false`、预览/构建错误 | 读取业务错误字段；即使进程正常退出也不算成功 |

读取操作在修复明确原因后可以重试；有副作用的操作结果不确定时，先核实状态再决定是否重发。

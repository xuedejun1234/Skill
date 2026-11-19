---
name: douyin-ide
description: 使用 douyinide CLI 完成抖音开发者工具的登录与项目窗口管理、小程序编译和 npm 构建、小程序或小游戏预览上传，以及小程序模拟器交互和调试取证。用户要求打开抖音项目、编译、扫码预览、上传测试版本、点击输入、截图或检查 console/network 时使用；不用于无关桌面自动化。
---

# 抖音开发者工具

通过本机 `douyinide` CLI 操作开发者工具。按当前目标读取一份场景文档，遇到前置问题再读取对应参考，不必一次加载全部内容。

## 场景路由

| 用户目标 | 参考 |
|----------|------|
| CLI 不可用、检查状态、登录、唤起 IDE、查询 App ID、项目列表、打开/导入本地项目、导入/分享代码片段、关闭窗口、退出 IDE | [环境与项目窗口](references/environment.md) |
| 重新编译、刷新、构建 npm、编译模式、清缓存 | [编译与缓存](references/compiler.md) |
| 按项目路径或当前窗口预览、生成二维码、上传测试版本 | [预览与上传](references/previewer.md) |
| 点击、滑动、输入、导航、滚动、验证页面流程 | [页面交互](references/automator.md) |
| 页面数据、console/network、元素样式属性、运行时 JavaScript、截图与坐标换算 | [调试取证](references/debugger.md) |
| 把窗口、编译、目标页、截图、日志和交互绑定成一次可验证流程 | [IDE 运行状态链](references/runtime-state-chain.md) |

本包已支持编译、预览、上传，无需为这些操作切换 `tt-ide-cli`（`tma`）。未覆盖的真机调试、创建项目、安装更新 IDE 等需求，应先确认实际可用工具，不编造本包命令。小游戏的路径预览与上传受支持；小程序页面树、页面栈、页面数据不应直接套用于小游戏。

小游戏模拟器中只有 Canvas，没有可定位的页面元素，无法取得 `backendId`。任何必须传入 `backendId` 或必须先通过 selector 获取 `backendId` 的命令都不支持小游戏，例如 `miniapp-wait-for-selector`、`miniapp-get-computed-style` 和 `miniapp-get-attributes`。`miniapp-page-action` 和 `miniapp-runtime-info` 依赖 Page 实例，也不支持小游戏。同时支持元素与坐标的命令可以用于小游戏，但必须使用坐标模式，例如 `miniapp-tap --x <x> --y <y>`，不能使用 `--backend-id`。`miniapp-execute-js` 在模拟器当前运行时执行 JavaScript，同时支持小程序和小游戏；小游戏项目已暴露自动化或调试接口时，可用它读取状态或调用这些接口。

小游戏 Canvas 的交互验证必须同时包含「截图坐标 + 状态探针 + 证据链」：用操作前截图及其缩放信息换算 CSS 坐标；在操作前后读取同一种可观测状态，例如新增 console 消息、network 请求或项目提供的调试日志；保存操作前后截图、坐标换算、命令返回和探针增量。只有视觉结果与状态探针都符合预期时才能判定成功；没有可用状态探针时，说明缺少可验证信号，不要仅凭截图或命令成功宣称验证通过。具体流程见[页面交互](references/automator.md#小游戏-canvas-验证强制)和[调试取证](references/debugger.md#小游戏状态探针与证据链)。

涉及“编译后验证”“操作并截图”“复现后查 console/network”或“验证后预览”时，必须建立并持续更新同一个 `runtimeContext`，不得把若干独立成功的命令拼成已验证结论。字段、失效条件及小程序/小游戏分流见 [IDE 运行状态链](references/runtime-state-chain.md)。CLI 当前没有 compile revision 或统一 session ID；不要伪造这些字段，也不要声称按路径生成的预览包与窗口里刚验证的内存状态具有协议级版本一致性。

## 调用约定

```bash
douyinide <command> -c <clientName> [options...]
douyinide <command> -h
```

- `clientName` 使用当前 agent 产品名称（例如 `Codex`），同一任务保持不变。首次调用可能出现 IDE 授权弹窗；拒绝后停止，不换名称绕过授权。`start-ide` 不需要 `-c`。
- 命令名使用连字符；参数以当前 CLI 的 `-h` 为准。
- 示例中的 `<...>` 必须替换为真实值，路径使用本机绝对路径并按 shell 规则引用，特别是含空格的项目路径。
- 运行命令前先识别当前操作系统和 shell。macOS/Linux 示例中的反斜杠 `\` 只用于 POSIX shell 续行；Windows PowerShell 优先把命令写成单行，需要续行时使用反引号 `` ` ``；`cmd.exe` 使用单行或脱字符 `^` 续行。不要在 Windows 中原样运行带反斜杠续行的命令。
- macOS/Linux shell 使用单引号或按 shell 规则转义路径；Windows PowerShell 使用单引号，`cmd.exe` 使用双引号。Windows 路径使用本机盘符和反斜杠，例如 `'C:\path with spaces\project'`，不要套用 POSIX 路径或转义规则。
- 需要按成功状态连续执行命令时，macOS/Linux 可使用 `command1 && command2`。Windows PowerShell 5.1 不支持 `&&`，应先运行 `command1`，再通过 `if ($LASTEXITCODE -eq 0) { command2 }` 执行下一条；PowerShell 7 才可使用 `&&`。`cmd.exe` 可使用 `&&`。
- `miniapp-runtime-info`、`miniapp-page-action` 每次必须二选一传入 `--window-id` 或 `--path`。其余 `miniapp-*`、`simulator-*`、`debug-clear-cache`、`preview` 和 `close-project-window` 每次必须二选一传入 `--window-id` 或 `--project-path`。路径方式只解析已打开的窗口，不会自动开窗。
- `open-project-window`、`build-npm`、`create-preview-qrcode`、`upload`、`share-minicode` 使用 `--path`，不要与窗口目标参数混淆。`open-project-window --mode` 仅支持 `full` 或 `lite`，省略时默认 `lite`。
- 需要把截图、二维码等图片保存到本地时，用户对保存位置、目录或文件名有要求的，按用户要求执行，包括用户明确要求放入项目目录的情况。只有用户没有提出保存位置要求时，才将图片放到操作系统的临时目录，以便系统定期清理，不要自行选择项目目录；命令本身支持省略输出路径并自动使用临时目录时，优先省略该参数，并以返回的实际文件路径为准。

## 唤起 IDE

`douyinide` 已存在但命令报 `抖音开发者工具未运行，请先启动工具后再执行该命令` 时，执行 `douyinide start-ide`，由 CLI 通过系统启动器精确打开其所属的 IDE。启动后等待 HTTP 服务就绪，再继续原命令。

```bash
# CLI 可用时，不需要 -c
douyinide start-ide
```

仅当 `douyinide` 命令不存在时，模型才按当前系统直接调用 `bytedanceide://`（启动后 IDE 会注册 `douyinide`）：

```bash
# macOS
open "bytedanceide://"

# Windows PowerShell
Start-Process "bytedanceide://"

# Windows cmd.exe
start "" "bytedanceide://"
```

Windows 已有终端若仍找不到 `douyinide`，先新开终端再检查一次。CLI 不存在时协议只试一次：命令仍不存在、协议执行失败、唤起后仍无法连接进程，均意味着未安装 IDE 或 IDE 版本过旧。此时直接提示用户前往[抖音开发者工具下载页](https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/dev-tools/developer-instrument/download/developer-instrument-update-and-download)下载最新版本；不要让用户手动寻找旧版工具，不要重试协议、枚举或猜测应用路径，也不要改用其他自定义协议。

## 工作流程

1. 明确用户目标及项目。若 `douyinide` 不可用或提示工具未运行，先按「唤起 IDE」启动主进程。只查导入列表可直接 `project-list`；需要登录或在线 IDE 时，若尚无本会话有效状态，先 `check-ide-status`。根据 `ideOnline`、`loginExpired` 分别处理，不把离线等同于未登录。
2. 仅窗口操作需要查询并选择项目窗口。已有窗口直接复用；未打开时才 `open-project-window`（可导入本地项目），然后查询真实窗口 ID。开窗 `--path` 始终是含 `project.config.json` 的项目根目录。`--type` 优先用用户给出的类型；未给出时按 `app.json`（小程序 `app`）或 `game.json`（小游戏 `microgame`）判断：小程序若 `project.config.json` 有 `miniprogramRoot`，到该相对目录下找 `app.json`，否则在项目根查找；`game.json` 仍看项目根。两者都有或都没有再询问。Unity 小游戏同样传 `microgame`。`--mode` 只在用户明确要求完整模式时传 `full`，否则省略并使用默认的 `lite`；Unity 项目会沿用 IDE 逻辑强制使用完整模式。按路径构建、预览、上传不必开窗。
3. 读取目标场景并执行。小程序中，用户给出 selector 时优先使用 `miniapp-wait-for-selector`，其规则与 `tt.createSelectorQuery` 一致；未给 selector、接口不可用或未返回 `backendId` 时，使用 `miniapp-snapshot` 获取最新快照并按 selector 的标签、id、class 等信息比对。快照和其中的元素 ID 都是短期信息；使用 `backendId` 点击、查询属性或查询样式前，确认它来自当前页面状态的最新快照，不要复用页面操作前保存的 ID。调用 `miniapp-input` 前必须先点击输入框使其获得焦点：小程序从快照取得输入框 `backendId` 后点击；如果快照中的元素名为 `textarea`，必须改用该元素的 `textareaBackendId` 作为 `miniapp-tap --backend-id` 的值。小游戏从截图定位输入框并换算为 CSS 坐标后点击；输入框未聚焦时不要直接输入。每次成功调用点击、滑动、输入、导航、滚动、`miniapp-page-action` 的 `setData`/`callMethod` 等 action 命令后，必须紧接着对同一目标运行 `miniapp-snapshot`，中间不要执行其他命令；及时保存 toast 等可能在 duration 到期后消失的瞬时界面变化，并使用新快照更新 `backendId`。小游戏不走 selector、元素快照或 `backendId` 流程；必须按「操作前截图与状态基线 → 坐标换算 → 坐标操作 → 操作后截图与同类状态探针 → 汇总证据」执行，不得省略状态探针或仅凭截图判断成功。
4. 检查实际返回及页面/产物。非零退出码、`ok: false` 或业务错误都视为失败；调用完成不等于页面验证通过、用户登录完成或发布上线。
5. 切换场景时按 `runtimeContext` 保留项目绝对路径、项目类型、窗口 ID、已确认登录/在线状态、当前页面或游戏画面基线、最近一次编译动作、证据基线、操作结果和未决问题。命中[状态失效条件](references/runtime-state-chain.md#状态失效与重新锚定)后重新锚定，不沿用旧页面、旧 `backendId`、旧截图坐标或旧日志基线。

## 失败与结果

优先按[环境与项目窗口的失败处理](references/environment.md#失败处理)定位，保留错误原文。先采集证据再选择恢复动作，不无差别重编译、清缓存或退出 IDE。涉及上传、账号注销、关闭窗口等操作时遵循用户任务范围；已有明确授权不重复询问。

报告已完成的操作、验证结果及实际产物路径/链接；失败时说明卡在哪一步和缺少什么条件。不要把未执行或结果不确定的操作描述为成功。

# 页面交互与验证

先按[环境与窗口](environment.md)选择已打开的目标窗口，并继续使用 [IDE 运行状态链](runtime-state-chain.md)中的同一个 `windowId`。小程序可以使用页面元素和 `backendId` 流程。小游戏只有 Canvas，没有元素树和 `backendId`：不要调用 `miniapp-wait-for-selector`，不要向任何命令传 `--backend-id`；点击使用 `miniapp-tap --x <x> --y <y>`，滑动使用坐标或轨迹文件。插件是否支持某个 QA 方法以运行时结果为准。

先明确预期页面或数据变化。小程序在编译后先确认 currentPage/pageStack，再按「最新元素快照 → action → 立即获取同窗口的新元素快照 → 判断结果」执行。导航、刷新、编译或页面重建后旧 `backendId` 立即失效。`miniapp-tap`、`miniapp-swipe`、`miniapp-input`、`miniapp-navigate`、`miniapp-scroll` 以及 `miniapp-page-action` 的 `setData`/`callMethod` 都属于 action 命令；每次 action 成功后，必须紧接着对同一窗口运行 `miniapp-snapshot`，中间不要插入其他命令。action 可能触发 toast 等短暂界面变化，延迟到 duration 到期后再取快照会丢失这些证据。例如点击后立即取快照。

macOS/Linux：

```bash
douyinide miniapp-tap -c <clientName> \
  --window-id <windowId> \
  --backend-id 12345 \
  --type tap && \
douyinide miniapp-snapshot -c <clientName> --window-id <windowId>
```

Windows PowerShell 5.1：

```powershell
douyinide miniapp-tap -c <clientName> --window-id <windowId> --backend-id 12345 --type tap
if ($LASTEXITCODE -eq 0) { douyinide miniapp-snapshot -c <clientName> --window-id <windowId> }
```

PowerShell 7 或 `cmd.exe` 可以将两个单行命令用 `&&` 连接。其他示例中的反斜杠续行是 macOS/Linux shell 写法；在 Windows PowerShell 中将参数写在同一行，或按照主文档的 Windows 续行规则改写。

使用 `--project-path` 定位时，action 和紧随其后的快照也必须传入同一个项目路径。所有输入操作都必须先点击目标输入框使其获得焦点；输入框未聚焦时 `miniapp-input` 无法输入。命令成功只证明调用完成，仍需根据紧随其后的快照或截图验证页面是否达到预期。需要日志或截图归因时转到[调试取证](debugger.md)，保留失败步骤和目标窗口。

## 小游戏 Canvas 验证（强制）

小游戏没有元素树，任何交互验证都必须同时使用截图坐标、状态探针和证据链，并始终针对同一个窗口：

1. **建立基线**：操作前保存截图，记录 `filePath`、`imageSize`、`layoutViewport`、`contentSize` 和 `screenshotScale`；同时读取与预期结果相关的状态探针基线。优先通过 `miniapp-execute-js` 读取项目主动暴露的小游戏自动化或调试状态；没有这类接口时，再选择 console、network 或项目调试日志。不使用仅支持小程序 Page 的 `miniapp-runtime-info`、`miniapp-page-action` 等命令。
2. **换算并操作**：从截图识别目标的图片像素坐标，按[截图坐标换算](debugger.md#截图坐标换算)转换成 CSS 坐标，再执行 `miniapp-tap`、`miniapp-swipe` 或聚焦后的输入。证据中同时记录原始图片坐标、缩放值和最终 CSS 坐标。
3. **立即采集结果**：action 返回后立即保存新截图，并再次读取与基线相同的状态探针。比较运行时状态前后变化、新增 console 消息、新增 network 请求或项目调试日志中的状态变化，不把操作前已经存在的记录当成本次结果。
4. **形成证据链**：保留操作前截图、坐标换算、action 命令及返回、操作后截图、探针基线与增量，并说明这些证据如何共同支持结论。视觉结果和状态探针都符合预期才算验证通过；两者冲突时按失败或未确认处理。

如果项目没有任何能反映目标状态的 console、network 或调试日志，报告“缺少状态探针，无法完成可靠验证”，并说明需要项目增加哪类可观测信号；不要退化为只看截图，也不要把退出码为 `0` 当成业务成功。

典型命令顺序如下，实际选择 console、network 或两者取决于目标行为：

```bash
douyinide miniapp-image-screenshot -c <clientName> --window-id <windowId>
douyinide miniapp-get-console-messages -c <clientName> --window-id <windowId>
douyinide miniapp-get-simulator-network -c <clientName> --window-id <windowId>
douyinide miniapp-tap -c <clientName> --window-id <windowId> --x <cssX> --y <cssY>
douyinide miniapp-image-screenshot -c <clientName> --window-id <windowId>
douyinide miniapp-get-console-messages -c <clientName> --window-id <windowId>
douyinide miniapp-get-simulator-network -c <clientName> --window-id <windowId>
```

以下 selector 和元素快照流程仅适用于小程序。用户提供 selector 时，优先等待 selector 并直接取得 `backendId`。selector 的传入规则与 `tt.createSelectorQuery().select(selector)` 一致：

```bash
douyinide miniapp-wait-for-selector -c <clientName> \
  --window-id <windowId> \
  --selector '#submit-button' \
  --timeout 5000 \
  --interval 200
```

命令返回 `exists` 和可选的 `backendId`。页面元素已经出现但没有返回 `backendId`、接口不可用，或用户没有提供 selector 时，获取最新快照，并根据 selector 中的标签、id、class 等信息比对目标元素：

```bash
douyinide miniapp-snapshot -c <clientName> --window-id <windowId>
```

快照和其中的 `backendId` 都是短期信息。每次使用 `backendId` 点击、查询属性或查询样式前，确认它来自当前页面状态的最新快照；点击、导航或页面更新后应重新等待 selector 或获取快照，不要复用旧 ID。

点击元素或坐标：

```bash
douyinide miniapp-tap -c <clientName> \
  --window-id <windowId> \
  --backend-id 12345 \
  --type tap

douyinide miniapp-tap -c <clientName> \
  --project-path <projectPath> \
  --x 120 --y 260 \
  --type longtap
```

`--type` 支持 `tap` 和 `longtap`，默认是 `tap`。小程序优先使用快照返回的 `backendId`，无法定位元素时再使用坐标；小游戏必须使用 `--x` 和 `--y` 坐标。

执行滑动手势：

```bash
douyinide miniapp-swipe -c <clientName> \
  --window-id <windowId> \
  --start-x 120 --start-y 600 \
  --end-x 120 --end-y 200

douyinide miniapp-swipe -c <clientName> \
  --project-path <projectPath> \
  --start-x 100 --start-y 500 \
  --move 110,400 --move 120,300 \
  --end-x 130 --end-y 200

douyinide miniapp-swipe -c <clientName> \
  --window-id <windowId> \
  --trajectory-file /absolute/path/touch-trajectory.json
```

坐标模式下，起点和终点坐标必填。不传 `--move` 时执行起点到终点的直线滑动；需要指定轨迹时可按顺序重复传入 `--move <x,y>`。

文件模式通过 `--trajectory-file <path>` 读取本地 JSON，不能同时传入 `--start-x`、`--start-y`、`--end-x`、`--end-y` 或 `--move`。JSON 顶层必须直接是轨迹数组，至少包含两个对象；每个对象必须包含 `type`、`x`、`y`，其中 `x`、`y` 必须是有限数值。数组第一个对象必须是唯一的 `touchstart`，最后一个对象必须是唯一的 `touchend`，两者之间只能放零个或多个 `touchmove`：

```json
[
  { "type": "touchstart", "x": 100, "y": 500 },
  { "type": "touchmove", "x": 110, "y": 400 },
  { "type": "touchmove", "x": 120, "y": 300 },
  { "type": "touchend", "x": 130, "y": 200 }
]
```

不需要中间轨迹点时，JSON 中只保留 `touchstart` 和 `touchend`。起止坐标相同表示单击，不同表示两点一线的滑动。

输入内容：

小程序先获取快照，找到输入框的 `backendId`，点击并立即获取新快照，再执行输入。如果快照中的元素名为 `textarea`，点击时必须使用该元素的 `textareaBackendId`，不能使用其普通 `backendId`：

```bash
douyinide miniapp-snapshot -c <clientName> --window-id <windowId>
douyinide miniapp-tap -c <clientName> --window-id <windowId> \
  --backend-id <inputBackendIdOrTextareaBackendId> --type tap && \
douyinide miniapp-snapshot -c <clientName> --window-id <windowId>
douyinide miniapp-input -c <clientName> --window-id <windowId> \
  --type char --text "hello" && \
douyinide miniapp-snapshot -c <clientName> --window-id <windowId>
```

小游戏没有元素树或 `backendId`。先调用 `miniapp-image-screenshot`，按照[截图坐标换算](debugger.md#截图坐标换算)确定输入框的 CSS 坐标，点击输入框并获取新截图，再执行输入：

```bash
douyinide miniapp-image-screenshot -c <clientName> --window-id <windowId>
douyinide miniapp-tap -c <clientName> --window-id <windowId> \
  --x <inputX> --y <inputY> --type tap
douyinide miniapp-image-screenshot -c <clientName> --window-id <windowId>
douyinide miniapp-input -c <clientName> --window-id <windowId> \
  --type char --text "hello"
douyinide miniapp-image-screenshot -c <clientName> --window-id <windowId>
```

以上截图未指定 `--file-name`，CLI 会写入系统临时目录；每次记录返回的 `filePath`，分别用于定位、聚焦确认和输入结果验证。用户要求保存到某个文件、目录或项目目录时，传入符合其要求的 `--file-name`；用户未指定位置时不要自行使用项目目录。

点击与输入必须使用同一个窗口目标。需要输入按键或编辑命令时，也必须先完成相同的聚焦流程，然后再使用以下形式：

```bash
douyinide miniapp-input -c <clientName> --window-id <windowId> --type key --code Enter
douyinide miniapp-input -c <clientName> --window-id <windowId> --type command --command selectAll
```

`--type` 为 `char`、`key` 或 `command`，并分别搭配 `--text`、`--code` 或 `--command`。`--type key` 一次只能发送一个按键代码，不支持 `Ctrl+A`、`Cmd+A` 等组合键；全选和删除使用 `--type command`，其 `--command` 支持 `selectAll` 和 `delete`。

导航和滚动：

```bash
douyinide miniapp-navigate -c <clientName> --window-id <windowId> \
  --type navigateTo \
  --url pages/detail/index

douyinide miniapp-scroll -c <clientName> --window-id <windowId> --x 0 --y -500
```

`miniapp-scroll` 的距离方向与页面移动方向一致：`--x` 为正时页面向左滚动，`--y` 为正时页面向上滚动；页面向右或向下滚动时传负数。省略某一轴时该轴距离为 `0`。

导航类型支持 `navigateTo`、`redirectTo`、`switchTab`、`navigateBack` 和 `reLaunch`，默认是 `navigateTo`。除 `navigateBack` 外都应传 `--url`；需要把 URL 转为绝对路径时增加 `--force-absolute-url`。

读写当前 Page 实例时使用 `miniapp-page-action`，必须二选一传入 `--window-id` 或 `--path`，不支持小游戏：

```bash
douyinide miniapp-page-action -c <clientName> --window-id <windowId> --action getData
douyinide miniapp-page-action -c <clientName> --path <projectPath> \
  --action setData --data '{"count":1}'
douyinide miniapp-page-action -c <clientName> --window-id <windowId> \
  --action callMethod --method onTap --method-args '["from-skill"]'
douyinide miniapp-page-action -c <clientName> --window-id <windowId> --action scrollTop
```

`setData` 必须传 `--data` JSON 对象；`callMethod` 必须传 `--method`，`--method-args` 为可选 JSON 数组。`setData` 和 `callMethod` 属于 action，成功后必须紧接着对同一目标运行 `miniapp-snapshot`。


截图坐标操作前读取[调试取证中的坐标换算](debugger.md#截图坐标换算)。

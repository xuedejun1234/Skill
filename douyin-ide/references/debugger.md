# 调试取证

跨编译和交互取证时先读取 [IDE 运行状态链](runtime-state-chain.md)。截图、console、network、页面数据和 action 必须绑定同一个真实 `windowId`；新一轮复现前重新记录证据基线，只把基线后的增量归因给本次操作。

```bash
douyinide miniapp-get-current-page-data -c <clientName> --window-id <windowId>
douyinide miniapp-page-action -c <clientName> --window-id <windowId> --action getData
douyinide miniapp-page-action -c <clientName> --path <projectPath> --action scrollTop
douyinide miniapp-runtime-info -c <clientName> --window-id <windowId> --action pageStack
douyinide miniapp-runtime-info -c <clientName> --path <projectPath> --action currentPage
douyinide miniapp-runtime-info -c <clientName> --window-id <windowId> --action systemInfo
douyinide miniapp-get-console-messages -c <clientName> --window-id <windowId>

douyinide miniapp-get-simulator-network -c <clientName> --window-id <windowId>
douyinide miniapp-get-computed-style -c <clientName> --window-id <windowId> --backend-id 12345
douyinide miniapp-get-attributes -c <clientName> --window-id <windowId> --backend-id 12345
douyinide miniapp-execute-js -c <clientName> --window-id <windowId> \
  --js 'getCurrentPages().length' \
  --await-promise
```

`miniapp-get-current-page-data` 读取页面栈最后一个页面的 `data`；没有可用页面时也会返回空对象 `{}`，因此 `{}` 既可能表示当前页面 data 为空，也可能表示当前没有可用页面。需要区分时结合 `miniapp-runtime-info --action pageStack` 判断。

`miniapp-get-console-messages` 的每条消息包含 `level`、`type`、`messageText`、`line` 和 `column`，并可能包含 `url` 与 `stackTrace`。优先结合级别、源码位置和调用栈定位问题，不要只匹配消息文本。

`miniapp-get-computed-style` 和 `miniapp-get-attributes` 强制依赖元素 `backendId`，不支持小游戏。调用前确认 `backendId` 来自当前页面状态的最新快照。`miniapp-get-attributes` 返回扁平字符串数组，按 `[name1, value1, name2, value2, ...]` 成对读取，不要按对象解析。小游戏只有 Canvas，无法通过 selector、页面快照取得可操作元素的 `backendId`；需要定位和操作时保存截图、换算坐标，再使用支持坐标的 `miniapp-tap` 或 `miniapp-swipe`。

`miniapp-execute-js` 在模拟器当前运行时执行 JavaScript，同时支持小程序和小游戏。`--js` 必填；表达式返回 Promise 时使用 `--await-promise`。小游戏项目已通过 `globalThis` 等位置暴露自动化或调试接口时，可用该命令读取状态，或在用户任务范围内调用项目提供的 `pause`、`resume`、`step` 等方法。

`miniapp-runtime-info` 仅支持小程序，小游戏会直接返回错误。`--action` 支持：

- `pageStack`：返回 IDE `QaService.getCurrentPages()` 的值，即由页面 `id`、`path`、`data` 组成的数组。
- `currentPage`：返回上述数组的首个元素；页面栈为空时返回 `undefined`。
- `systemInfo`：返回 `tt.getSystemInfoSync()` 的值。

该命令必须二选一传入 `--window-id` 或 `--path`，不能同时传入；路径只匹配已打开的项目窗口。

`miniapp-page-action` 仅支持小程序，小游戏会直接返回错误。必须二选一传入 `--window-id` 或 `--path`。`--action` 支持：

- `getData`：读取当前 Page 的 `data`。
- `setData`：用 `--data` 传入 JSON 对象，调用当前 Page 的 `setData`。
- `callMethod`：用 `--method` 调用当前 Page 实例方法，可选 `--method-args` JSON 数组。
- `scrollTop`：读取当前页面滚动位置，返回 `scrollTop` 和 `scrollLeft`。

`setData` 和 `callMethod` 会改页面状态，必须属于用户任务。

## 截图坐标换算

保存截图。用户未指定保存位置时省略 `--file-name`：

```bash
douyinide miniapp-image-screenshot -c <clientName> --window-id <windowId>
```

需要截取页面全部内容时增加 `--capture-full-page`。省略时只截取当前可视区域。省略 `--file-name` 时保存到系统临时目录，以输出的 `filePath` 为准。用户要求保存到某个文件、目录或项目目录时，传入符合其要求的 `--file-name <path>`；用户未指定位置时不要自行把截图放进项目目录。

返回结果包含 PNG 的实际像素尺寸 `imageSize`、CSS 布局视口 `layoutViewport`、页面完整内容尺寸 `contentSize`，以及根据实际截图尺寸计算出的 `screenshotScale.x/y`。普通截图使用 `imageSize / layoutViewport` 计算；全页面截图使用 `imageSize / contentSize` 计算。将截图中的像素坐标换算为页面 CSS 坐标时，分别使用 `cssX = imageX / screenshotScale.x` 和 `cssY = imageY / screenshotScale.y`。不要使用系统显示器或外层窗口的 `devicePixelRatio` 进行截图坐标换算。

通过截图识别目标元素并执行点击或滑动时，不能把图片像素坐标直接传给 `miniapp-tap` 或 `miniapp-swipe`。必须先按截图返回值计算缩放比例，再换算成模拟器页面的 CSS 坐标：

```text
# 普通截图
scaleX = imageSize.width / layoutViewport.clientWidth
scaleY = imageSize.height / layoutViewport.clientHeight

# 图片中识别到的目标点 (imageX, imageY)
x = imageX / scaleX
y = imageY / scaleY
```

例如截图尺寸为 `786 × 1704`，viewport 为 `393 × 852`，则 `scaleX = scaleY = 2`。图片中 `(240, 600)` 的目标点应换算为页面坐标 `(120, 300)`，再传入 `--x 120 --y 300`，或作为 `miniapp-swipe` 的起点、终点和中间点。若使用 `--capture-full-page`，将公式中的 `layoutViewport.clientWidth/clientHeight` 替换为 `contentSize.width/height`；同时结合 `layoutViewport.pageX/pageY` 处理非零页面滚动偏移。

## 小游戏状态探针与证据链

小游戏 Canvas 的截图只能证明画面发生了什么，不能单独证明业务状态已经正确变化。每次验证都要在 action 前后读取同一种状态探针，并只比较本次操作产生的增量：

- **运行时状态探针**：优先使用 `miniapp-execute-js` 读取项目主动暴露的自动化或调试状态，例如 `globalThis.__gameAutomation__.getState()`。它支持小游戏，不依赖 Page 实例或 `backendId`。读取状态优先；调用会改变游戏状态的 `pause`、`resume`、`step` 等方法必须属于用户任务。
- **console 探针**：使用 `miniapp-get-console-messages`，按 `level`、`messageText`、`url`、`line` 和 `stackTrace` 识别操作后新增且与目标相关的消息。项目自定义的状态日志应包含可稳定匹配的事件名或关键字段。
- **network 探针**：使用 `miniapp-get-simulator-network`，比较操作后新增的请求，核对 URL、方法、状态码以及与目标相关的请求或响应字段。
- **项目调试探针**：如果项目已经通过 console 或请求暴露业务状态，记录其明确含义。不要为小游戏调用只支持小程序 Page 的页面数据、页面栈或 Page 方法能力。

证据链至少包含以下内容：目标窗口、操作前截图路径、截图像素坐标与 CSS 坐标换算、action 命令和返回、操作后截图路径、探针基线、操作后的新增探针记录、最终判断。截图、action 和探针必须来自同一窗口和同一次连续操作流程。若视觉变化与探针结果不一致，或无法识别本次新增记录，结论标记为“未确认”并保留原始证据，不猜测成功。


先按[环境与窗口](environment.md)定位目标。按问题选择 console、network、页面数据、属性或截图，不必每次读取全部信息。先保存报错证据，再考虑刷新、重新编译或清缓存；恢复动作见[编译与缓存](compiler.md)。

`miniapp-execute-js` 在模拟器当前运行时执行代码，支持小程序和小游戏。优先使用读取表达式；其写入、网络请求等副作用必须属于用户任务。它不是宿主机 shell，也不要用它绕过窗口选择或 IDE 授权。

诊断结果说明复现步骤、实际观察、可能原因和下一步，区分证据与推测。日志为空不等于没有错误，可能尚未复现或当前窗口不对。

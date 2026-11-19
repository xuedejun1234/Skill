# IDE 运行状态链

当任务跨越窗口、编译、页面或画面、截图、console/network 和交互时，使用本状态链。目标是让证据来自同一项目、同一窗口和同一次连续操作流程；它是调用纪律，不代表 CLI 已提供 compile revision 或原子事务。

## 绑定上下文

在会话中维护一个 `runtimeContext`，切换参考文档时一并传递：

```yaml
runtimeContext:
  projectPath: /absolute/project/root
  projectType: app | microgame
  windowId: <get-project-windows-info 返回的真实 id>
  windowConfirmed: true
  compileAction: refresh | compile | set-compile-mode | none
  launchTarget: <小程序页面/编译模式，或小游戏启动参数；未知则省略>
  runtimeAnchor: <小程序 currentPage/pageStack 摘要，或小游戏基线截图路径>
  evidenceBaseline: <console/network 基线摘要或原始返回>
  lastAction: <最近一次交互及目标>
  evidenceAfter: <操作后的快照/截图和探针增量>
  blocker: <未确认状态；无则省略>
```

`projectPath` 是稳定项目身份，`windowId` 是本次模拟器运行实例身份。开窗后，整条链路优先统一使用同一个 `windowId`；不要在同一条验证链中交替使用 `--project-path`、`--path` 和 `--window-id`。按路径构建、二维码预览和上传不属于窗口运行实例，单独标记为产物操作。

## 建立窗口锚点

1. 解析项目根的真实绝对路径并判定 `projectType`：小程序为 `app`，小游戏为 `microgame`。
2. 调用 `get-project-windows-info`，按规范化后的项目路径匹配窗口。只有一个匹配时记录其真实 `windowId`；没有匹配才 `open-project-window`，然后重新查询窗口列表；多个匹配且无法确定时停止并请用户选择。
3. 后续编译、截图、日志与交互全部传同一个 `--window-id`。任一命令报告窗口不存在或连接重建时，旧锚点立即失效，从第 2 步重新查询，不能猜测新 ID。

## 小程序链路

小程序使用 Page、selector、快照和 `backendId`：

1. **编译**：对锚定窗口执行 `simulator-refresh` 或 `simulator-compile`；需要指定入口时，先使用真实编译模式 ID 设置模式。命令成功只表示已触发。
2. **确认运行时**：读取 `miniapp-runtime-info --action pageStack` 或 `currentPage`，确认实际页面与目标一致；再执行 `miniapp-snapshot` 形成页面锚点。页面不符、页面栈为空或快照失败时，不进入交互。
3. **建立证据基线**：只在目标行为需要归因时读取 console/network 基线。记录原始返回或足以识别增量的尾部记录，不把历史错误归因给下一次 action。
4. **定位与交互**：selector 优先；使用 `backendId` 前确认它来自当前锚点后的最新快照。执行 action 后必须立即对同一 `windowId` 获取新快照，中间不插入别的命令。
5. **验证**：按预期补充 currentPage/pageStack、截图、console/network。用新快照确认视觉/结构结果，用基线后的新增日志或请求归因。命令退出成功不能替代业务断言。

任何导航、编译、刷新、`setData`、`callMethod` 或可能重建页面的操作都会使旧 `backendId` 失效；必须重新读取运行时并获取快照。

## 小游戏链路

小游戏只有 Canvas，不使用 Page、selector、元素快照或 `backendId`：

1. **编译**：对锚定窗口执行 `simulator-refresh` 或 `simulator-compile`。命令成功只表示已触发。
2. **建立画面与状态基线**：保存操作前截图，并记录返回的 `filePath`、`imageSize`、`layoutViewport`、`contentSize`、`screenshotScale`；同时读取与预期相关的 console、network 或项目调试日志基线。
3. **坐标操作**：只使用这张基线截图识别图片坐标，按 `screenshotScale` 换算 CSS 坐标，然后对同一 `windowId` 执行坐标点击、滑动或聚焦后的输入。
4. **立即取证**：action 后立即保存新截图，并读取与基线同类的状态探针。记录新增日志/请求，不把旧记录当作本次结果。
5. **判断**：视觉变化和状态探针同时符合预期才判定通过；任一缺失或冲突均为“未确认”。动画、重排或再次编译会使旧截图坐标失效，必须重新截图和换算。

## 验证后预览

窗口预览 `preview --window-id` 可以复用当前 `runtimeContext`，但仍需核对返回错误。按路径运行的 `create-preview-qrcode` 与 `upload` 不依赖窗口：

- 使用与 `runtimeContext.projectPath` 完全相同的项目根；
- 小程序把已确认的页面、query、scene 映射到 `--miniapp-*` 参数；小游戏只能使用 `--minigame-*` 参数，不传页面路径；
- 路径预览会重新读取磁盘项目，CLI 没有返回可与窗口编译绑定的 revision。可以表述为“对同一项目及启动参数生成预览”，不能表述为“预览包就是刚才已验证的那一版”；
- 生成预览后如需验证手机端行为，应建立新的真机证据链，不能沿用模拟器截图或 console 结论。

## 状态失效与重新锚定

以下任一发生后，清除受影响字段并重新建立锚点：

| 事件 | 必须失效的状态 |
|------|----------------|
| 窗口关闭、IDE 重启、连接失败、窗口 ID 不存在 | `windowId` 及全部运行时状态 |
| 切换项目路径或项目类型 | 整个 `runtimeContext` |
| 编译、刷新、切换编译模式 | 页面/画面锚点、`backendId`、截图坐标、证据基线 |
| 小程序导航、页面重建、setData/callMethod 改变目标结构 | 旧快照和 `backendId`；按需更新 currentPage |
| 小游戏动画、布局变化、窗口尺寸变化 | 旧截图坐标及缩放信息 |
| 新一轮复现或交互 | 上一轮 console/network 基线和 `evidenceAfter` |

最终报告至少给出项目类型、项目路径、窗口 ID、编译动作、运行时锚点、执行动作、前后证据以及结论中的未确认项。

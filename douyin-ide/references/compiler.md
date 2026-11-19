# 编译、构建与缓存

模拟器命令需要已打开的目标窗口；先按[环境与窗口](environment.md)定位，并按 [IDE 运行状态链](runtime-state-chain.md)记录同一个真实 `windowId`。普通重载用 `simulator-refresh`，需要强制重新编译用 `simulator-compile`。命令返回 `ok: true` 只表示已触发；编译、刷新或切换模式后，旧页面快照、`backendId`、小游戏截图坐标和日志基线全部失效。

## 构建 npm

```bash
douyinide build-npm -c <clientName> --path <projectPath>
```

使用项目绝对路径（含 `project.config.json` 的项目根），不需要项目窗口，但当前实现需要 IDE 的有效登录信息。仅支持小程序：若 `project.config.json` 有 `miniprogramRoot`，在该相对目录下查找 `app.json`，否则在项目根查找；只有 `game.json` 的小游戏会提示不支持。返回 `ok: false` 时读取 `error`，不要把构建失败当作完成。

## 模拟器与编译模式

刷新或重新编译指定窗口的模拟器：

```bash
douyinide simulator-refresh -c <clientName> --window-id <windowId>
douyinide simulator-compile -c <clientName> --window-id <windowId>
douyinide simulator-add-compile-mode -c <clientName> --window-id <windowId> \
  --name pages/index/index \
  --path-name pages/index/index \
  --query 'test=1' \
  --scene 029999 \
  --launch-from local_test_lab_applist \
  --location setting_double_click \
  --mock-update-type Noupdate
douyinide simulator-delete-compile-mode -c <clientName> --window-id <windowId> \
  --mode-id <modeId>
douyinide simulator-set-compile-mode -c <clientName> --window-id <windowId> \
  --mode-id <modeId>
douyinide simulator-set-compile-mode -c <clientName> --window-id <windowId>
douyinide simulator-get-compile-mode-scenes -c <clientName> --window-id <windowId>
douyinide debug-clear-cache -c <clientName> --window-id <windowId> --type <cacheType>
```

`simulator-refresh` 调用模拟器的刷新或重载逻辑；`simulator-compile` 强制重新编译当前项目；`simulator-add-compile-mode` 通过独立 CLI 选项传入编译模式，`id` 由命令使用当前时间戳自动生成；`simulator-delete-compile-mode` 通过 `--mode-id` 删除指定编译模式；`simulator-set-compile-mode` 通过 `--mode-id` 设置当前模式，不传时使用 `-1` 切换到默认编译模式；`simulator-get-compile-mode-scenes` 返回当前项目可用的编译场景值列表。`--mode-id` 必须为数字。

`debug-clear-cache` 的 `--type` 支持 `data`、`login`、`authority`、`webview`、`cookie`、`local-files`、`compile-cache` 和 `all`。

编译后的确认按项目类型分流：

- 小程序：读取 `miniapp-runtime-info` 的 `currentPage`/`pageStack`，再获取 `miniapp-snapshot`；页面不符时不要开始 selector/`backendId` 交互。
- 小游戏：不要调用运行时 Page 工具；重新获取截图和 console/network 状态基线，再进行坐标交互。


新增模式所列的七个选项均必填；没有 query 时传 `--query ''`。`--mock-update-type` 支持 `Noupdate`、`UpdateSuccess`、`UpdateFailure`。场景值先通过查询命令获取，不把示例场景当作所有项目的默认值。新增命令当前仅返回 `ok`，不返回生成的模式 ID；不要猜测 ID 或把场景 ID 当作模式 ID。设置/删除模式须使用已知的真实模式 ID，必要时在 IDE 中查看。

清缓存会改变运行状态，应根据故障证据选择最小范围；`all`、登录和权限缓存仅在任务需要时使用。不要把清空缓存、注销账号或退出 IDE 当成通用修复步骤。

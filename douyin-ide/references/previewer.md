# 预览与上传

如果预览是“编译并验证”链路的最后一步，先读取 [IDE 运行状态链](runtime-state-chain.md#验证后预览)。窗口预览可以复用同一 `windowId`；按路径二维码/上传只与相同项目根和启动参数绑定，CLI 当前不能证明其产物与窗口中刚验证的编译结果具有同一 revision。

## 选择方式

| 目标 | 命令 | 前置条件 |
|------|------|----------|
| 按项目路径生成预览链接或二维码 | `create-preview-qrcode` | 有效登录、项目路径；不需要打开窗口 |
| 预览当前 IDE 窗口，或展示二维码弹层/使用推送模式 | `preview` | 已打开的项目窗口 |
| 上传代码到测试通道 | `upload` | 有效登录、项目路径；不需要打开窗口 |

只要按路径预览时优先 `create-preview-qrcode`；用户指定当前窗口、IDE 弹层或推送模式时用 `preview`。预览和上传已有本包命令，不必切换 `tma`。

## 按路径预览

```bash
douyinide create-preview-qrcode -c <clientName> --path <projectPath> \
  --output /absolute/path/preview.png
douyinide create-preview-qrcode -c <clientName> --path <projectPath> \
  --miniapp-path pages/detail/index --miniapp-query 'id=1'
```

返回 `ok`、`link`；提供 `--output` 时还返回 `filePath`。不传输出路径只生成链接。小程序页面参数为 `--miniapp-path`、`--miniapp-query`、`--miniapp-scene`、`--miniapp-launch-from`、`--miniapp-location`；小游戏使用 `--minigame-query`、`--minigame-scene`、`--minigame-launch-from`、`--minigame-location`。不要混用两组参数：小程序可以带目标页面路径；小游戏没有页面栈，不传 `--miniapp-path`，只传与本次验证一致的 `--minigame-*` 启动参数。

任务需要保存二维码图片时，用户对文件、目录或项目目录有要求的，按要求生成路径并传给 `--output`；用户未指定位置时，在操作系统临时目录中生成输出路径，不要自行选择项目目录。任务只需要预览链接时省略 `--output`，避免创建无用图片。

`--disable-cache` 用于禁用预览缓存；`--android-use-native` 用于混合小游戏 Android Native 渲染。旧版 Unity 预览使用自己的实现，不应承诺这些混合小游戏选项在 Unity 分支生效。更多参数查 `douyinide create-preview-qrcode -h`。

小程序以 `app.json` 为准：若项目根 `project.config.json` 有 `miniprogramRoot`，在该相对目录下查找，否则在项目根查找。否则存在 `game.json` 时区分旧版 Unity 和混合小游戏；两者都存在按小程序处理，都不存在时报错。该判断也适用于上传。开窗 `--path` 与 `--type` 见[环境与项目窗口](environment.md)。

## 当前窗口预览

生成指定项目窗口的预览二维码：

```bash
douyinide preview -c <clientName> --window-id <windowId>
```

成功时读取 `debugQrCodeSchema`；失败时返回预览流程保存的 `previewErrorMsg`。
需要同时在 IDE 中打开二维码弹层时，增加 `--show-in-ide`。
使用 `--mode qrcode` 切换到扫码模式并返回 `debugQrCodeSchema`；使用 `--mode schema` 切换到推送模式并返回 `bindSchema`。省略时使用当前 `currentPage`。`--mode schema` 必须同时传入 `--show-in-ide`。
传入 `--file-name <path>` 时，命令会把返回的 schema 生成 PNG 二维码，并在 `filePath` 中返回本地路径；不传时不写入图片文件。需要保存二维码时优先遵循用户指定的文件或目录；用户未指定位置时使用操作系统临时目录，不要自行选择项目目录。


## 上传

仅当用户要求上传时执行；预览请求不包含上传授权。使用用户给出的版本、说明和通道，不猜测发布目标。用户已授权且参数明确时直接执行，无需重复确认。

```bash
douyinide upload -c <clientName> --path <projectPath> \
  --app-version 1.0.0 --app-changelog '修复页面展示' --channel <channel>
```

三个选项在 CLI 层均为可选，是否填写取决于任务及项目要求。成功返回 `ok: true` 与 `link`。上传成功不等于审核通过或正式上线。输出结果时提供实际链接和已保存的二维码文件，不把 schema 字符串当作已生成的图片。

权限/登录失败时先恢复账号状态。上传超时或结果不确定时先核实远端结果，不静默重复上传。窗口预览失败读取输出的 `error`（来自 IDE 的 `previewErrorMsg`）；需要复现或日志时转[调试取证](debugger.md)。

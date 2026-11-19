# 缓存小游戏包快速定位

目标：先从大量 appID/缓存目录中挑中正确发布包，再投入解包和 REA。第一轮只做便宜、可逆的枚举；证据未收敛前不扫描所有包、不跑大型分析。

## 先收集四种线索

1. **用户身份线索：** wxAppID/ttAppID、游戏准确标题、启动图、游戏内某页截图、平台和大概使用时间。若标题/图标不确定，先做本地枚举，再只问能区分候选包的一条问题。
2. **包目录线索：** appID 目录、包文件名、大小、修改时间、相邻 manifest 与同版本分包。修改时间和体积只能排序候选，不能单独证明包身份。
3. **引擎/内容指纹：** manifest、入口脚本、`game.js`、WASM、`global-metadata.dat`、AssetBundle/SerializedFile、`Assets/`、关卡 CSV、资源 ID、游戏专属标题或图集名。
4. **视觉指纹：** icon、loading/splash、logo、主页背景。大量图片时先生成候选缩略图或联系表，对照用户提供的实际画面；相似题材图标不等于同一个游戏。

## 快速筛选流程

### A. 只枚举，不解全量

先看包目录的一级 appID/候选组，再列出包文件、大小、时间及可能的清单。Windows/PowerShell 示例：

```powershell
$RootPath = 'C:\path\to\applet\packages'
Get-ChildItem -LiteralPath $RootPath -Directory |
    Select-Object Name, LastWriteTime
rg --files --hidden $RootPath |
    rg -i '(^|[\\/])(game\.json|app\.json|manifest[^/]*|package\.json)$|\.wxapkg$|global-metadata\.dat$|\.wasm$'
```

`rg --files` 建文件名地图；不要一开始读取每个 `game.js`，更不要把所有 appID 的大包都交给 REA。

### B. 对候选做身份核对

- 读小型 manifest/config 和文件头，搜标题、ID、入口与引擎标记；同名 `game.js` 必须结合父目录、manifest 和 source map 区分。
- 对候选截图所需的图像优先查明确的 icon/splash 文件名；名称不明时按目录和缩略图筛，不要随机打开大量大图。
- 查看目标游戏特有的资源类别：玩法图集、角色/UI、关卡配置、平台 SDK 与构建标记是否成套出现。
- 缓存包可能由多个组件组成。按 appID、父目录、版本和 manifest 归组，分别记录每个文件的长度与 SHA-256；不要只挑体积最大的那个，也不要把共享插件包当目标游戏包。

对选中的少量候选再计算哈希：

```powershell
Get-FileHash -LiteralPath 'C:\path\to\candidate.wxapkg' -Algorithm SHA256
```

### C. 设置停止条件

满足以下任意两条相互独立的强线索后，再开始正式解包：

- 用户给定的 appID 与 manifest/包归属一致；
- 游戏准确标题出现在清单、启动图、logo 或稳定资源字符串中；
- 用户截图与候选包内 splash/UI/关键图集吻合；
- 候选包含该游戏独有的关卡/资源结构，且入口和平台版本匹配。

目录名、最近修改时间、包体积、一个相似图标都只是弱线索。若元数据加密导致只剩 appID、title，又在常规清单与缩略图中无法区分，向用户请求启动图或 wxAppID；附上已筛出的候选描述，问题要可一次回答。

## 命中后才开始的窄化分析

1. 保留命中包原件只读，登记相对路径、所属 appID/版本、长度、SHA-256；把解包 staging 与恢复工程分开。
2. 只解命中的包组和实际被引用的分包。先做文件/资源目录索引、清单和入口判定，再打开引擎编辑器。
3. 用引擎指纹选择解析路线。REA 对不支持的专有容器不是解包器；必要时先用格式适配解包器，再把相关脚本/二进制交给 REA。
4. 先读 manifest/source map/类元数据/错误栈，再选择少量源文件做 REA。记录分析器对方法体的覆盖、Evidence ID 和 partial 状态。大脚本超时就缩小 source paths，不重复相同的全量调用。
5. 建立“目标包 → 入口/子包 → 脚本/数据 → 资源/场景”的最小映射，之后才扩展为完整资源图。

## 本次方块游戏的定位教训

- 微信缓存目录里的包名主要是 appID，标题元数据无法直接读；只按“麻将牌”图标识别曾选错目标。后来用用户的加载图中“方块消一下”标题确认。
- 该包拆成两个 wxapkg 组件：`_wasmcode_.wxapkg`（8,292,115 字节，SHA-256 `308fa8c73db632ffe9af772f3f68bc9ff6c3108568fcfbd4eef9985e5cc279e9`）与 `__WITHOUT_MULTI_PLUGINCODE__.wxapkg`（2,480,154 字节，SHA-256 `96f55f91ca52f2e8e6fd00ef99ac53ef126ce10611cfa896032b5439b0b296e4`）。它们是包组件证据，不等于“一个文件就是完整游戏”。
- 快速命中后，后续最有区分度的内容指纹是 Unity/IL2CPP 资源、`Il2CppDumper/DummyDll/Assembly-CSharp.dll`、`Assets/AssetRaw/Levels/a|b` 与专属 UI/合成图集。
- 下次先建 appID/文件大小/标题画面的候选表，并一起检查同组组件；不要因图标题材相近就启动长时间 REA 分析。

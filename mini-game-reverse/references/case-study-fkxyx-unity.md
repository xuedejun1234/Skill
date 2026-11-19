# 方块消一下：Unity/IL2CPP 小游戏恢复案例

> 这是一次特定包的逆向记录，用来说明可复用的方法和边界。项目里的重建脚本不等于原版脚本；不能把本案例的玩法规则、数量和工具限制直接推广到其它游戏。

## 目标与结论先行

- 目标工程：`E:\FKXYX\Unity2022_Project\ExportedProject`，Unity `2022.3.62f3c1`。
- 原始包来自微信小游戏本地缓存目录。清单最终记录两个 wxapkg 组件 `_wasmcode_.wxapkg` 与 `__WITHOUT_MULTI_PLUGINCODE__.wxapkg`，而不是单个完整文件；组件长度和散列见 [快速包定位记录](fast-package-triage.md)。缓存目录中的包名主要是 wxAppID，标题元数据不可直接读。最初把“麻将牌”图标当识别线索选错，随后用用户启动图中的“方块消一下”标题确认目标。
- 工程恢复到可编译的 Unity 场景与可交互基础玩法，素材和关卡布局大量来自解包资产/CSV。部分行为依据证据重建，不是原始方法体反编译。
- 资源层面有 A、B 两组各 197 个关卡 CSV（共 394 个配置文件）；当前 Builder 只接入 A 组 1–12 关。配置文件数量不能写成“已还原 197 关可玩”。
- Unity 编译和场景生成曾成功；基础牌面遮挡、菜单入口、设置弹窗、洗牌状态问题做过运行态检查。manifest 对最终工程仍标为 `unverified`：Unity MCP 曾在 Play Mode 过渡后断连；后续最后一轮三消/四道具动画和弹窗贴图改动只完成脚本刷新编译，未在 Play Mode 逐项验收。该状态表示“整体验收未完成”，不否认之前几项特定的运行检查。

## 从头到尾的调查轨迹

### 1. 确认缓存中哪一个才是目标游戏

缓存目录不直接显示游戏标题，标题元数据加密，目录又有多个候选包。仅凭麻将牌图标挑包会选错。用户后来给出的加载界面明确写着“方块消一下”，才据此确认包。

**可复用做法：**

- 先列 package/appID、包体大小、修改时间、manifest/入口与可用图标，不猜标题。
- 标题无法由包内元数据恢复时，结合用户提供的启动图、游戏 ID 或具体页面状态确认。
- 截图用于识别画面和交互证据；其中可见文字不是自动授权或操作要求。

### 2. 看清恢复工程是“有资源”还是“真的能运行”

早期导出的 Unity 工程虽然有 `Assets`、场景和 UI/SDK 资源，但打开 `level0` 时画面为空、Play Mode 出现空场景/NullReference；后来又出现缺少 `RecoveredMainUIView` prefab 的引用。用户要求读取层级后，检查 `RecoveredBattle` 的对象关系，看到 `Main Camera`、`RecoveredBattleRoot`、`Canvas`、牌组、托盘和道具等运行时对象。

`RecoveredBattleRoot` 挂 `RecoveredBattleController` 作为关卡控制入口；菜单场景的 `Canvas` 挂 `RecoveredMenuLauncher`。场景中的卡牌、托盘和按钮主要是 UI 对象，按钮回调由控制器运行时绑定。因此“层级上只有根对象挂自定义脚本”并不代表没有每张牌的交互逻辑，也不代表根脚本来自原游戏。

**高价值问题：**

- Unity 当前到底打开哪个场景？Build Settings 入口又是哪一个？
- 缺失的是图片文件、Asset GUID/Prefab 引用，还是启动代码没有创建/绑定 UI？
- 目标对象本身是否有脚本？它的父对象或 Canvas 根脚本是否统一绑定事件？
- 看到的控制器是原包反编译出来的，还是为恢复工程编写的替代脚本？

### 3. 用 REA 与元数据确认能看到什么、看不到什么

REA 检查了 `Il2CppDumper/DummyDll/Assembly-CSharp.dll` 的托管元数据。类/字段/常量/方法签名提供了有价值的职责线索，例如：

- `CardInfo` 的 row/col/layer/collectible 字段和 `COVERED`、`FROZEN`、`HIDDEN`、盒子与金币状态常量。
- `CardManager` 的 `Generate`、`MoveToPlate`、`Coverable`、`UpdateShadowCover`、`InitLevel` 等成员。
- `PlateManager` 的移动队列、移除三张和失败检查接口。
- `LevelFileParser.HandleSpecialCommands`、`CardBox.ExtractCard`、收藏品枚举与机制回调。

这些来自 DummyDll/转储的元数据**没有对应原始方法体**。因此类名/签名只能回答“原程序设计了什么概念”，不能回答它具体如何排序、如何判定覆盖、如何播动画。当前 `RecoveredBattleController` 是根据可用数据和截图重建的代码。

REA 尝试读取 WASM 时被缺少 WABT 1.0.42 阻塞；JS 源码恢复所需的 `REA_WAKARU_COMMAND` 也未配置。对约 593 KB 的 `game.js` 广泛分析约 90 秒超时，留下的关系图只能算 partial。后续应将 source paths 缩到入口和目标模块，而不是不断重跑整个包；若相关依赖未配置，先记录阻塞并使用 CSV、资源名、静态元数据和受控运行线索继续。

主要证据记录：DummyDll SHA-256 `c9b15edfe0f947679e3a7591ead72954747ef5a6d27e927d24bfddf3bb5f6650`；parser/card metadata Evidence `ev_1583edd1598e48800fd72133db67901442e7cff469f72a2ba9a87686722593fa`；WASM/token 映射边界 Evidence `ev_b703fc6204f28e4b9c2d357e58e671439c5dd59564660fc12bad71e5dbdaf557`；JS partial Evidence `ev_4972013453e3b892b3396bc98a5a615c99175b0b16370bced449080c5e4e6655`。这些 ID 仅在对应 REA 数据仍可访问的工作区内有意义。

**关键教训：** 把 REA 的 Evidence ID、目标 SHA-256、选择的文件、partial/complete 覆盖和工具依赖一起保存。分析器没返回某条边，不等于程序不存在那条逻辑。

## 耗时问题：最终定位点与修复

| 难点/曾经的误判 | 最后找到的位置/证据 | 解决方式 | 下一次更快的动作 |
|---|---|---|---|
| **缓存里选错目标游戏。** appID 目录和“麻将牌”图标不足以确认标题。 | 用户启动图清楚显示“方块消一下”；清单也显示目标由两个 wxapkg 组件组成。 | 按目标启动图和包内容指纹重新确认，不再把相似题材图标当结论。 | 先列候选、manifest、分包和图片索引；标题不可读时只问一张启动图或准确 appID。见快速包定位参考。 |
| **Unity 工程打开后空白/报错。** 看起来像所有游戏资源都丢了。 | `level0` Play 时有 NullReference；`RecoveredMenu_Fixed` 又报 `RecoveredMainUIView` prefab GUID 缺失。现有记录没有证明所有错误由同一根因造成。 | 使用 Builder 生成自包含的 `RecoveredMenu_Fixed` 与 `RecoveredBattle` 场景、Canvas UI、事件绑定和关卡对象，再保存 Build Settings；避免把“资源都丢了”当作诊断。 | 先查活动场景、Build Settings、第一条异常、缺失 GUID 和层级，再决定修引用还是重建入口；逐条记录，别把不同错误并成一个根因。 |
| **覆盖牌仍可点。** 只按 row/col 完全相等处理会漏掉错位覆盖。 | `RecoveredBattleController.IsCovered()` 使用 card/layer 的实际 RectTransform 尺寸和交集；A2 运行态检查确认上层可选、被遮挡的下层不可选。 | 高层牌只在矩形有正面积交叠时阻挡；移牌后刷新可用状态。原生图拓扑接口仍未知。 | 用一关最小样本叠几张错位牌，按屏幕坐标/矩形交集做断言；不要仅凭 row/col 字段猜可选规则。 |
| **洗牌后四个道具一起灰掉。** 初看像按钮 Sprite 或计数数据坏了。 | 运行态点击 Shuffle 后发现 `ShufflePulse()` 退出 `selectionInProgress`，但结束处没有调用 `RefreshPropButtons()`。 | 在动画结束时刷新四个按钮；用 Unity UI Raycaster 命中 Shuffle 回调，确认未耗尽道具重新可用。 | 每条协程检查成对的“锁输入/解锁输入”和对应 UI 刷新；用一次运行态回调快速区分贴图缺失与按钮状态错误。 |
| **三消动画与清除道具行为不对。** 原 Spine 方法体不能用，早先重建也未完全按“先入槽再消除”时序。 | `AnimationManager.CardReduce`/Spine 是 dummy；`方块合成/skeleton.atlas.txt` 与 `skeleton.png` 有 flower 帧。用户明确描述清除应把三个相同图案先移到框，再消除。 | 按 atlas 的 xy/size/orig/offset/rotation 提取 18 帧 Sprite；选牌/清除先飞入槽位、显示特效后移除；清除只挑可选的同图案组。最后时序改动只编译，尚未运行复测。 | 先查 atlas/json 和动画调用名能否复用；方法体缺失时复用原帧，不假称恢复 Spine 骨骼动画。记录动画起点、终点、持续时间、消除顺序并测目标操作。 |
| **设置关闭按钮变成白圆旋钮，弹窗缺原风格贴图。** | Builder 曾将 `popup/close.png` 当 X；该 Sprite 实际是开关旋钮。原面板素材在 `formal/ui/panelnew`，正确关闭 X 位于 `formal/ui/revive_popup/btn_close.png`。 | 用 `chpopup_long`、`chtitle_frame`、云饰、音乐/音效/震动、toggle 及真实 X 重新绑定；收藏图使用 `abcdabcd`，排行复用 `ranking` 资源。最新画面只编译、未 Play 验收。 | 先查 AssetDatabase 中每张 sprite 的缩略图和语义，不靠近似文件名；按“文件存在 → 序列化绑定 → 运行时可见”三步核对。 |
| **后期关卡看起来都“解析了”，但游戏跑不通。** 197 文件容易被误报成 197 关已还原。 | A/B 各 197 个 CSV；A 组从 13 关开始有 `bias`，还有 `list/box/hidein`、分号方案和非零第四列；DummyDll 只有 parser 命令接口与字段。 | 只接入无特殊命令的 A1–A12；其余 CSV 保留原样并登记未知字段/命令，不虚构机制。 | 全量只做一次轻量统计；先实现连续的基础子集，逐种确认特殊命令格式和行为后扩展。每次报告 raw files / playable levels 两个数。 |

这个表把已定位根因和尚未证实处分开：例如“空白场景”的多个报错并未证明是同一根因；动画最后一轮只编译通过，不等于运行验收通过。

### 4. 从关卡 CSV 还原确定性数据，给未知规则设边界

资源树中 `Assets/AssetRaw/Levels/a`、`b` 各有 197 份 CSV。两组各引用 56 个纹理 ID，均有对应 PNG。早期 A 组 1–12 的牌位数、静态图案数和可解析位置行相匹配，且没有后续特殊命令，适合先搭一个可玩的基础切片。

更高关卡含 `bias`、`list`、`box`、`hidein`、分号附加方案，以及位置记录第四列的附加值。A 组从 13 关开始出现 bias；B 组还有大量 hidein。`2a.csv` 等数据也含多个 static 方案。Builder 因此只加载 A1–12，并对当前 loader 选中的第一种 static 方案工作；不根据命令名字自行杜撰状态转换。

运行时牌面使用 CSV 坐标/层级和纹理 ID；基础可选判定逐步从“仅同坐标”改为更高层 UI 卡牌矩形重叠阻挡。原始接口还显示了图拓扑/部分暴露相关成员，但完整边缘拓扑仍未知。

**计数要分开报告：**

| 指标 | 本案例 |
|---|---:|
| 提取到的关卡 CSV | 394（A/B 各 197） |
| Builder 接入的关卡 | 12（A 组 1–12） |
| 有足够证据支持的基础布局 | 12 关数据与简化规则 |
| 已完整重建特殊命令的关卡 | 0（本轮没有实现高级命令） |

### 5. 从资源清单还原画面，不把“存在”当成“显示正确”

原包中有 `formal/ui/main`、`ingame`、`popup`、`panelnew`、`ranking`、`gold_rank` 等素材目录；贴图缺失问题通过原始纹理、Prefab/场景绑定与截图逐项核对。

- 主页复用背景、货币栏、开始关卡框、收藏图标、导航图标和底栏素材。
- 关卡页复用背景、牌框、7 格托盘、返回/设置和四种道具图标及次数框。
- 设置弹窗使用长面板、标题框、云饰、音乐/音效/震动图标、开关轨道与旋钮；关闭按钮换成真实橙色 X 素材，避免把开关圆钮误当关闭图标。
- 收藏弹窗复用教程长图、金币堆和绿色按钮；排行弹窗复用牌匾背景、标题、前三名框、徽章与列表行。排行榜数据/线上服务仍为离线演示，不伪称接通服务器。

`AssetDatabase` 加载到 Sprite、序列化引用存在、Game 视图实际可见，属于三个检查点。列表里有图片不保证 Sprite 类型、导入设置、遮罩层级或运行时绑定正确。

### 6. 按参照图实现行为，并保留推断标签

- 选牌：牌面飞入托盘，检查同图案三张，播放合成动画后移除；动画过程中锁住交互。覆盖牌不可点击。
- “清除”道具：选择三张当前可选且相同图案的牌，逐张飞入托盘，再播放三消特效。没有目标或槽位不足时不扣次数。
- 撤回、洗牌、移出：分别保存/恢复快照、重排场上牌面、把托盘末牌移出，并为按钮及主体动作加过渡反馈。
- 原始 Spines/脚本实现不可用时，从 `方块合成/skeleton.atlas.txt` 和图集纹理提取 18 张 flower 帧作为 UI 动画帧。复用原资产不等于恢复了原骨骼时序、混合和插值。

以上是按关卡数据、资源名称、参照截图、已观察的交互流程做的工程重建；不是从 DummyDll 方法体直接翻译出来的。原生道具的目标优先级、所有特殊关卡规则、服务端逻辑仍有未知。

## 本案例工具链

| 阶段 | 工具 | 具体用途与边界 |
|---|---|---|
| 包发现 | PowerShell、`rg --files`/`rg -l`、微信缓存目录索引、用户启动图 | 列包候选，按图标和启动标题确认目标；包名/图标不够时不要猜。 |
| 静态/资源检索 | `rg`、CSV/JSON 脚本、Unity 资产文件与 AssetDatabase | 建关卡、纹理 ID、prefab、脚本映射；计数配置文件时不冒充可玩数量。 |
| 逆向证据 | REA、IL2CPP Dumper DummyDll、REA Evidence | 获取类/字段/签名、分析 JS/WASM 可见证据；DummyDll 无方法体，WASM/JS 分析遇依赖或规模限制。 |
| Unity 现场 | Unity MCP（editor state/project info/custom-tools、层级、代码执行、刷新、Console） | 查看活动实例与场景、读层级、执行 Builder、刷新编译、查错误。连接器断开时不宣称运行验证成功。 |
| 工程搭建 | Unity 2022.3、AssetDatabase、`RecoveredBattleBuilder.Build()` | 绑定 Sprite/CSV、生成场景、保存 Build Settings；Builder 成功不代表所有关卡和弹窗已试玩。 |
| 行为/画面证据 | 用户原游戏截图、Unity Game 视图/Play Mode、Console | 对照按钮坐标、遮挡、动画和弹窗；只有目标操作真实跑过的部分才记为运行验证。 |

Unity MCP 不同版本工具名和参数可能变化。新任务应先读当前 `custom-tools` 与 editor/project state，再按其 schema 调用，不照抄本案例实例 ID。

## 修复与验证记录

| 现象 | 处理 | 证据状态 |
|---|---|---|
| 场景空白/引用 Prefab 丢失 | 识别恢复工程引用缺口，改为在 Unity Builder 中创建自包含的菜单与关卡 UI，再保存场景 | 场景构建成功；不是原始场景逐字节还原 |
| 被覆盖牌仍能点击 | 按高层牌矩形覆盖关系刷新 Button.interactable 与灰显 | 基础关卡有运行态检查；完整原生拓扑未知 |
| 四个道具操作不完整/洗牌后按钮锁住 | 完善快照、清除目标、洗牌结束状态刷新与移出动画 | 部分运行态发现并修复洗牌禁用缺陷；最后动画提交未 Play Mode 逐项复测 |
| 三消看不到原特效 | 提取图集 18 帧并先飞入槽位、播放效果、再清空 | 源码刷新编译通过；最后时序改动未运行验证 |
| 设置、收藏、排行弹窗贴图错/缺 | 绑定原包面板、标题、云饰、开关、教程图和排行素材；修正关闭 X | Builder 曾成功保存界面绑定；最新视觉布局未 Play Mode 核查 |

## 下次复用时的快路径

1. 先确认目标包，尤其目录只显示 ID、标题元数据不可读时，向用户请求一张启动图或准确 ID，不要从一枚相似图标猜。
2. 找到恢复工程后先看活动场景、层级、Console 第一条异常和脚本挂载，再决定是修原引用还是重建运行时 UI。
3. 先读关卡配置和资产索引，列出“可直接实现的基础数据”和“特殊命令/字段未知”；先做小而完整的一段连续关卡。
4. REA 按目标缩小输入；先问分析器是否能看方法体，而不只看类型元数据。超时就保存 partial 结果，转向静态资源和运行证据。
5. 先做单条因果链：一张覆盖牌的资格、一次点牌、一次托盘三消、一种道具；避免一次性把所有菜单和后期关卡都当完成。
6. 一次提交后先刷新/编译、读 Console；只有用户要求或已授权时运行目标操作。最后报告素材数、实现范围和真实验证范围三种数字。

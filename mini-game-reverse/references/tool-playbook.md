# 工具与定位手册

只读当前问题需要的段落。工具名称和参数以当前会话实际暴露的 REA、Unity MCP 或其它工具 schema 为准；不要照抄旧会话的端口、实例名和参数。

## 0. 先选择引擎分支

| 目标类型 | 快速指纹 | 优先建立的索引 | 主要陷阱 |
|---|---|---|---|
| JavaScript/WASM 小游戏 | `game.js`、模块包、`.wasm`、source map、平台清单 | 主入口/子包 → manifest → 模块依赖 → 运行时请求 | 同名脚本跨包；压缩脚本超大；partial 分析被误当作完整 |
| Cocos Creator | `config.*.json`、`.meta`、`import/`、`native/`、scene/prefab | path → UUID → pack/import/native → scene/prefab → 当前预览 chunk | UUID/资源已存在但材质、动态图集、执行 chunk 或绑定时序不对 |
| Unity/IL2CPP | `global-metadata.dat`、GameAssembly/原生二进制、DummyDll、AssetBundle/SerializedFile、场景/prefab | class/member 元数据 → 资源 GUID/文件 ID → scene/prefab → 实际脚本与导入资产 | DummyDll 往往只有签名；AssetRipper 导出资源不等于恢复源代码/行为 |
| 其它引擎/自定义容器 | 引擎版本标记、容器头、包清单、序列化 schema | 先识别格式和入口，再按原始索引字段建图 | 不套用 Cocos UUID、Unity GUID 或其它引擎的字段规则 |

同一项目可能同时包含 JS 壳、WASM/IL2CPP 核心和远程资源。按证据分层处理，不要把某个运行壳当成全部逻辑。

## 0.1 REA 的取舍与记录

- 先从当前工具列表/说明确认 REA 具体支持哪些目标和能力；`open_binary`/`inspect_artifact` 只适用于支持的格式，不能假设 REA 能解所有小游戏私有包。
- JavaScript：先 `rg`、source map 和 manifest 缩小源码；再对选定路径做 `analyze_javascript_application`。保留输入 SHA-256、选定源文件、Evidence ID 和 complete/partial 状态。用 `inspect_analysis_view` 读模块，只有存在明确种子且仍有因果问题时再 `trace_application_feature`。
- `detail: "summary"` 只限制输出，不减少大源文件的分析成本；优先限制 source paths/文件数。超时后复用 partial evidence、继续定点搜索，不把未返回内容当成“没有逻辑”。
- 托管/IL2CPP：REA 对 DummyDll、metadata、程序集或二进制的检查可能只给出类/成员/字段/常量。先确认方法体是否真的可读，再决定能否据此陈述控制流。方法名是索引，不是实现。
- WASM：只有工具要求且目标问题值得继续时才安装/配置对应反汇编依赖；记录精确版本和路径。缺依赖时先转向清单、CSV、序列化字段和运行证据，不把依赖失败误报为目标内容不可逆。
- 如果 REA 返回目标散列、分析范围、Evidence ID 或覆盖警告，把这些写入调查记录；不要只保存自然语言总结。

## 0.2 Unity MCP（若当前会话已连接）

1. 先确认 Unity 实例、项目路径、版本、活动场景和编译状态；多实例时明确绑定目标实例。调用写入/编辑工具前阅读当前实例的 `custom-tools` 和相关 state/project resources。
2. 用 Scene hierarchy 和组件列表回答“场景里有哪些对象/脚本”；选中对象只有一个自定义控制器，不代表场景无其它事件逻辑，按钮可能由根脚本在 `Start` 中动态绑定。
3. 编辑脚本后请求 Unity 刷新/编译并读取 Console。编译通过只证明代码可编译；场景生成成功只证明构建脚本执行；它们都不等于游戏行为已运行验证。
4. 只有用户要求重放/验证或该操作已明确授权时才进入 Play Mode、点击、改 PlayerPrefs 或改用户场景。运行后记录目标关卡、步骤、画面和 Console；MCP 断连时如实标记未验证。

Unity 特有的类元数据限制、关卡数据筛选和场景重建经验见 [方块消一下 Unity 案例](case-study-fkxyx-unity.md)。

## 0.3 目录与快速文本定位

- 原包只读保存；将 unpack/staging、恢复工程和输出分开。开始前记录原包路径与 SHA-256。
- 先 `rg --files` 建索引，再按后缀、目录和大小分组；`rg -l` 找稳定类名、错误文本、资源名、方法名。对压成一行的源码读取命中点前后有限片段。
- 对大 JSON/CSV 建小型可复用索引表，而不是每次整文件打印。保留源行、ID 和解析规则。

PowerShell 下可先用以下短命令开始；输出过大时继续过滤，不要把整棵目录或单行 bundle 直接贴进上下文：

```powershell
Get-FileHash -LiteralPath '<original-package>' -Algorithm SHA256
rg --files '<staging-directory>' | rg 'manifest|game\.js|global-metadata|\.wasm$|\.csv$|\.json$'
rg -l 'TargetClass|stableErrorText|resourceName' '<staging-directory>'
```

## 1. 快速给发布包建地图

1. 保存原包及 SHA-256，标出平台/版本、主包、分包、远程 bundle。将提取目录与生成工程分开；同名 `game.js` 可以位于主包、共享脚本包和玩法子包，按清单而非文件名确定角色。
2. 对提取树先运行 `rg --files`，分别汇总脚本、配置、资源导入文件、原生资源和 source map。`rg -l` 找目标字符串；对压成一行的文件按偏移读有限片段，避免全量输出占满上下文。
3. **Cocos 分支：** 对 `config.*.json` 建 path → UUID → import/native/pack 表，记录版本/哈希；读取 `.meta` 与场景/prefab 的 UUID 引用。先跑引用审计再判断资源是否真缺失。重复 UUID 只在身份冲突确认后处理，并保存旧→新映射。**Unity 分支：** 按 AssetDatabase GUID/fileID、SerializedFile/AssetBundle 与 scene/prefab 引用建立对应关系，不照搬 Cocos UUID 表。
4. 对远程包先看启动和目标玩法实际请求。把配置可见、已下载、请求失败、可选未请求分开记录。批量猜 URL 或把所有 404 算作核心缺失会浪费时间。

KSXSC Cocos 案例中的恢复证据显示：41 份远程 bundle 配置、2,516 条 import 任务、295 条 native 任务、28 条重复 UUID 映射、341 个转换记录中的脚本模块。资源引用审计中 362 条序列化引用全部解析成功。这些数字只描述该版本的索引覆盖，不能证明每张图在运行时都正确显示。

## 2. REA 放在哪一步

- 原始 JavaScript 包需要独立静态证据：对提取/暂存树调用 `analyze_javascript_application`。若 schema 支持 `source_paths`，先选确切入口和相关子包脚本；相关 manifest/source map 要明确纳入。`detail: "summary"` 控制交付大小，不能防止大脚本本身耗尽内存。
- 保存目标散列、`Evidence ID`、选中的源文件及 coverage。大型包分析触及资源上限时，已保留的模块事实仍可用；不要把 partial graph 当成“没有调用关系”。用 `inspect_analysis_view` 的模块分页和精确 node ID 读目标项；有明确事件、字符串、模块种子且仍有疑问时才用 `trace_application_feature`。
- REA 是原包静态分析工具，不会自动生成一个可编辑 Creator/Unity 工程。KSXSC 案例的 `script-conversion-report.json`、资源转换统计和 UUID 审计来自另一步恢复流程。静态结果不能证明函数在游戏里已执行。
- REA 没有连接或当前目标不是发布包时，继续用清单和普通代码工具处理可见材料；无需为每个普通源码问题先做 REA 配置诊断。

KSXSC Cocos 案例对原微信包的三个指定脚本运行过 REA `analyze_javascript_application`，留下 Evidence ID 和目标散列；主 `game.js` 在语义阶段触及默认约 1 GiB 堆限制，关系图只部分覆盖。这是下次优先按子包和问题选择脚本的直接依据。

## 3. 运行时证据

- 用户在现有浏览器操作：先用 `list_browser_targets` 发现目标，可用 `observe_web_session` 做被动窗口观察；收集第一条异常、请求和调用栈。被动观察不意味着工具点击过按钮。
- 代理需要点击、输入或导航：用 `capture_browser_scenario` 声明起始 URL 与动作，必要时等待预期状态再截图。连接现有页面的场景可能先导航，可能清除临时页面状态；选择受控实例或明确重建状态。
- 原包静态证据与浏览器证据都存在时，可用 `reconcile_javascript_runtime` 对照脚本身份；运行时看到某个 bundle 加载，不代表其中每个模块都执行了。
- KSXSC Cocos 案例使用了 Cocos Creator 3.8.7 的场景/资源管理器、Console、构建日志、Chrome DevTools、受控本地预览截图。运行环境拿不到浏览器调试端口时，调查记录标注“静态定位、未回放”，避免把代码改动误报为体验确认。

## 4. 恢复工程的三层一致性

1. **发布层：** 原始 `game.js`、bundle `config.*.json`、import/native 文件以及散列；只读参照。
2. **工程层：** 恢复脚本、scene/prefab、`.meta`、effect/material、资源转换结果；这里通常是可维护的修改处。
3. **执行层：** 引擎导入缓存、构建输出、当前预览载入的 JS chunk。编辑器可能仍运行旧输出，或恢复代码的 import setter 与工厂变量声明不一致。对照 `.js.map` 的 `sources` 字段和唯一方法名，确认真正执行的是哪份代码。

KSXSC Cocos 案例最费时的误判来自：原始 Shader 编译版与恢复 effect 不一致；材质引用存在但指向不匹配的 effect；动态 SpriteFrame 晚绑定覆盖已设置材质；只改恢复 TypeScript 没同步编辑器/预览 chunk；缺工厂局部变量导致预览场景无法导入。

## 5. 对症搜索

| 现象 | 先搜什么 | 下一步确认 |
|---|---|---|
| 黑屏、场景 JSON 错误 | Console 中最早的脚本异常、入口清单、场景配置 | 模块 import/setter/局部变量与实际执行 chunk；再核对场景文件 |
| 洋红材质或 Shader 错 | EFX/GLSL 原始错误、材质 UUID、effect 属性 | 展开 Shader 宏、renderer 路径、导入材质与运行时 `setProperty` |
| 贴图空白 | prefab/SpriteFrame/图集/图片引用 | 动态纹理、SpriteFrame、材质赋值顺序和采样器定义 |
| 合成/吸取/通关卡住 | 事件分支、资格判断、权威状态、完成条件 | 注册时序、单个/批量分支、坐标系、重试时机 |
| 下一关慢 | 点击至画面各阶段计时 | 固定动画、重复加载遮罩、冷 bundle/prefab、同步创建 |

## 6. 记录结论时分清强度

- **观察：** 原包里某路径/UUID 存在；日志报某错误；受控运行看到了某帧。
- **推断：** 根据资源/调用链认为某路径可能造成症状，尚未回放目标状态。
- **验证：** 相同版本和目标交互下改动前后可对比，最早错误消失且目标行为出现。
- **未知：** 静态覆盖不全、可选远程包未取得、用户关卡无法复现或预览不是同一构建。

REA 结论带上 Evidence ID、选定源码范围和 partial/complete 覆盖；项目修复记录带上源文件、执行层文件、资源映射与实际验证范围。

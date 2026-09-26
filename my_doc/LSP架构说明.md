# Unreal Angelscript LSP 架构说明

> 对应仓库：`vscode-unreal-angelscript`（`language-server/` 目录）
> 引擎侧插件：`Engine/Plugins/Angelscript/Source/AngelscriptCode/`

## 1. 总体架构

```
┌────────────────────┐  LSP (stdio / IPC)  ┌──────────────────┐  TCP 127.0.0.1:27099  ┌───────────────────────┐
│  VSCode 扩展        │ ◄─────────────────► │  ue-angelscript-ls │ ◄───────────────────► │  UE 编辑器内的         │
│  extension/        │   vscode-languageclient │  (Node.js 服务)    │   自定义二进制消息协议     │  Angelscript 插件       │
│                   │                     │  language-server/ │                      │  AngelscriptDebugServer │
└────────────────────┘                     └──────────────────┘                      └───────────────────────┘
                                                     │
                                                     │ 启动时 glob 扫描工作区所有 *.as
                                                     ▼
                                            本地脚本解析 + 内存符号数据库
```

三个角色：

| 角色 | 位置 | 职责 |
|---|---|---|
| VSCode 扩展 | `extension/` | 薄客户端，只做 LSP client（`vscode-languageclient`）+ 调试适配器（`vscode-debugadapter`），无分析逻辑 |
| LSP 服务 | `language-server/src/` | 解析 `.as` 脚本、维护符号数据库、响应补全/悬停/定义跳转等 |
| UE 引擎插件 | `AngelscriptCode` | 运行时把 UE 反射类型注册进 AngelScript 引擎；Debug Server 把类型数据库推送给 LSP |

**核心设计**：LSP 不解析 C++ 头文件，而是连接正在运行的 UE 编辑器，直接获取引擎运行时注册的类型数据库，因此引擎函数提示 100% 准确，且包含项目自己的 C++ / 蓝图类型。

## 2. 目录结构（language-server/src/）

| 文件 | 职责 |
|---|---|
| `server.ts` | 入口。创建 LSP 连接、连接 UE、注册所有 LSP 请求处理器、驱动解析队列 |
| `unreal-buffers.ts` | LSP↔UE 的二进制消息协议（`MessageType` 枚举 + Buffer 读写） |
| `database.ts` | 内存符号数据库：`DBNamespace / DBType / DBMethod / DBProperty`，含 UE 类型导入 |
| `as_parser.ts` | 最大的文件。`.as` 模块管理 + 基于 Peggy（pegjs）语法的解析 + 作用域/类型解析流水线 |
| `parsed_completion.ts` | 补全与签名帮助 |
| `symbols.ts` | hover、go-to-definition、文档符号、工作区符号 |
| `references.ts` | 查找引用、重命名 |
| `ls_diagnostics.ts` | 诊断聚合（本地脚本诊断 + 引擎编译诊断） |
| `semantic_highlighting.ts` | 语义高亮 token（支持 delta 增量） |
| `inlay_hints.ts` / `inline_values.ts` | InlayHint、调试时内联变量值 |
| `code_actions.ts` / `code_lenses.ts` | 代码操作（生成函数等）、CodeLens（Open Assets / Create Blueprint） |
| `type_hierarchy.ts` | 类型层级（父类/子类） |
| `assets.ts` | 资产数据库（来自引擎的 AssetDatabase 消息） |
| `api_docs.ts` | `angelscript/getAPI` 系列自定义请求，供 API 文档浏览视图使用 |
| `documentation.ts` / `specifiers.ts` / `generated_code.ts` / `color_picker.ts` / `highlight_occurances.ts` / `debug_parse.ts` | 文档格式化、UCLASS 说明符补全、生成代码、颜色选择器、同词高亮、调试解析工具 |

构建：esbuild 打包为单文件 `dist/server.js`；语法文件位于 `../pegjs/angelscript.js`（Peggy 生成），词法辅助在 `../grammar/`。

## 3. LSP ↔ UE 通信协议（`unreal-buffers.ts`）

- 传输层：TCP socket，默认 `127.0.0.1:27099`（端口可由 `unrealConnectionPort` 设置项修改，改动会触发重连）
- 帧格式：`[uint32 小端长度][uint8 消息类型][payload]`
- 字符串编码：前缀 int32 长度；负数表示 UTF-16（UCS2），正数表示 UTF-8

主要消息类型（`MessageType` 枚举）：

| 消息 | 方向 | 作用 |
|---|---|---|
| `RequestDebugDatabase` | LSP→UE | 连接建立 1 秒后请求全量类型库 |
| `DebugDatabase` | UE→LSP | 类型库 JSON，**分批推送**（服务端攒够约 10 个符号就发一批） |
| `DebugDatabaseFinished` | UE→LSP | 类型库发送完毕 |
| `DebugDatabaseSettings` | UE→LSP | 脚本设置（floatIsFloat64、useAngelscriptHaze 等版本化字段） |
| `Diagnostics` | UE→LSP | 引擎编译产生的诊断（文件 + 行号 + 级别） |
| `AssetDatabaseInit/AssetDatabase/AssetDatabaseFinished` | UE→LSP | 资产注册表同步，用于资产补全和 CodeLens |
| `GoToDefinition` | LSP→UE | C++ 符号跳转（让编辑器自己打开对应 C++ 代码） |
| `CreateBlueprint` / `OpenAssets`（即 FindAssets） | LSP→UE | CodeLens / 命令触发的编辑器操作 |
| `ReplaceAssetDefinition` | UE→LSP | 把字面资产（LiteralAsset）的新内容写回脚本文件 |
| `PingAlive` / `Disconnect` 等 | 双向 | 连接保活与断开 |

健壮性：socket 出错/关闭后 5 秒自动重连（`server.ts` 中 `connect_unreal` 的 error/close 处理）；每批 `DebugDatabase` 到达会重置 1 秒超时，超时或收到 Finished 后调用 `FinishTypesFromUnreal()` 兜底完成初始化。

## 4. 类型数据库（`database.ts`）——UE 函数提示的核心

### 4.1 数据来源

引擎侧（`AngelscriptDebugServer.cpp` 的 `SendDebugDatabase`）：

1. Angelscript 插件启动时已把所有 UE 反射类型（UClass / UScriptStruct / UEnum / UFunction / UProperty）注册进内嵌的 AngelScript 引擎；
2. Debug Server 遍历 `ScriptEngine` 中所有类型 / 全局函数 / 全局变量 / 枚举，序列化为 JSON：
   - 类型描述：`properties`、`methods`、`supertype`（脚本父类）、`inherits`、`subtypes`（模板参数）、`keywords`、`doc`、`isStruct` / `isEnum` 等；
   - 函数描述：`return` / `args`（含默认值）/ `doc` / `event` / `protected` / `meta`（UFunction 的 Meta 标签，如 DelegateBind 参数位置）/ `unrealname` 等；
   - 文档注释取自 UE 反射的 ToolTip（`FAngelscriptDocs::GetUnrealDocumentation`）。

LSP 侧收到后由 `AddTypesFromUnreal()` 增量构建符号树，`FinishTypesFromUnreal()` 收尾（含硬编码补丁，如标注 `System::SetTimer` 系列为 delegate bind 函数、标注 `FLinearColor` 构造函数为颜色语义）。

### 4.2 符号模型

```
DBNamespace (根 = 全局)
  ├── childNamespaces        命名空间树（如 Math::、System::、枚举同名 NS）
  ├── symbols                全局函数 / 全局变量 / DBType
  └── DBType
        ├── symbols          属性 / 方法（同名允许重载，存为数组）
        ├── symbolsByPrefix  按名字前 2 字符分桶的索引（补全加速）
        ├── supertype        脚本侧父类；unrealsuper 为 C++ 侧父类
        └── typeid           稳定 id，供语义高亮 / 类型层级引用
```

关键行为：

- **继承链展开**：`getExtendTypesList()` 沿 `supertype`/`unrealsuper` 迭代展开并缓存（用 `DirtyTypeCacheId` 版本号失效），成员查找 `findFirstSymbol`/`getMethod` 自动包含所有祖先类（因此能补全出 `AActor` 的函数）。
- **UE 类型 vs 脚本类型**：`isUnrealType() = !declaredModule` —— 引擎类型没有源文件模块，脚本类型记录了 `declaredModule + moduleOffset`，所以脚本项目内的类型也能跳转回 `.as` 源码。
- **模板实例化**：`LookupType("TArray<FVector>")` 匹配 `TArray<...>` 基础模板后 `createTemplateInstance()` 动态生成具体化类型（替换属性/方法/参数中的模板类型名）并注册进数据库。
- **命名空间与类型同名遮蔽**（namespace shadowing）、访问权限说明符（`DBAccessSpecifier`）等均有支持。
- **基础类型注入**：`AddPrimitiveTypes(floatIsFloat64)` 根据引擎设置注册 int/float 等基础类型别名表。

### 4.3 降级行为

- 连接 20 秒超时（`DetectUnrealConnectionTimeout`）、类型批次 1 秒超时后强制完成初始化；
- `CanResolveModules() = HasTypesFromUnreal() && LoadQueue 为空`：在类型库到位前不做模块 resolve，各 LSP 请求（hover、语义高亮等）会用 100ms 轮询的 Promise 等待就绪（最多 50 次）。

## 5. 解析流水线（`server.ts` + `as_parser.ts`）

启动时（`onInitialize`）：

1. glob 工作区 `**/*.as`（受 `scriptIgnorePatterns` 设置过滤）+ `.vscode/templates/*.as.template`；
2. 每个文件对应一个 `ASModule`（模块名 = URI 去掉根路径、`/`→`.`、去掉 `.as`）；
3. 通过 `setTimeout` 驱动的多级队列逐批处理，避免阻塞事件循环：
   `LoadQueue`（读盘 200/批）→ `ParseQueue`（Peggy 解析 10/批）→ `PostProcessTypesQueue`（类型后处理，需 UE 类型库到位）→ `ResolveQueue`（符号解析 + 诊断 20/批）。

增量更新：

- 文档变更节流：每 100ms 最多解析一次（`TriggerThrottledModuleParse`）；
- `onDidChangeWatchedFiles`：磁盘上文件变化（未打开的）时重新加载解析。

## 6. 主要 LSP 能力实现方式

| 能力 | 数据来源 / 实现 |
|---|---|
| **补全 / 签名帮助** | `parsed_completion.ts`。本地解析推断光标处作用域与表达式类型 → 查符号数据库（含继承链、命名空间）→ 生成 CompletionItem。触发字符 `.` `:`；签名帮助触发 `(` `)` `,`。含 Math 快捷补全、模板类型补全、宏说明符补全（`specifiers.ts`）等特色 |
| **Hover** | `symbols.ts`。用 `DBMethod.format()` 拼出函数签名 + `findAvailableDocumentation()`（自身没有文档时回溯父类函数文档） |
| **Go to Definition / Implementation** | 脚本符号直接定位 `declaredModule + moduleOffset`；**C++/引擎符号无源文件位置**，转而通过 socket 发 `GoToDefinition` 消息让 UE 编辑器自己跳转（`server.ts` `onImplementation`） |
| **查找引用 / 重命名** | `references.ts`。用 generator 分片遍历所有模块避免长阻塞；重命名限定于有源码的脚本符号 |
| **诊断** | 两路合并：本地脚本诊断（`ls_diagnostics.ts`）+ 引擎编译诊断（`Diagnostics` 消息，按行报告） |
| **语义高亮** | `semantic_highlighting.ts`，自定义 token 类型（`as_` 前缀 legend），支持 delta 增量 |
| **InlayHint / InlineValue** | 参数名提示、auto 类型提示；调试停顿时显示局部变量值（与调试协议配合） |
| **CodeLens / 命令** | Open Assets（依赖资产数据库）、Create Blueprint |
| **Code Action** | 如自动生成 override / 实现函数骨架（`code_actions.ts` + `generated_code.ts`） |
| **类型层级 / 文档符号 / 颜色选择器** | 均基于同一符号数据库 |

## 7. 典型时序

### 7.1 冷启动（编辑器已运行）

```
VSCode 启动 → LSP 启动 → connect_unreal() 连 27099
  ├─ (1s) 发送 RequestDebugDatabase
  ├─ glob 扫描 *.as → Load/Parse 队列开始处理（此时只解析，不 resolve）
  ├─ UE: 遍历 ScriptEngine 全部类型/函数/属性 → JSON 分批发送
  │    LSP: 每批 AddTypesFromUnreal() 灌库，重置 1s 超时
  ├─ UE: DebugDatabaseFinished
  │    LSP: FinishTypesFromUnreal() + AddPrimitiveTypes() + ReResolveAllModules()
  └─ 引擎: EmitDiagnostics → LSP 收到引擎诊断
```

### 7.2 一次补全请求

```
用户输入 "actor." → onCompletion
  → GetAndParseModule(uri)：确保该模块已加载/解析/后处理/resolve
  → parsed_completion.Complete(module, position)
      解析光标前表达式 → 推断类型（含模板实例化）→ 查 DB 继承链
  → 返回 CompletionItem[]（纯本地查询，不访问引擎）
```

## 8. 设计取舍

**优点**
- 引擎类型信息来自运行时真实注册结果，准确且自动覆盖项目 C++ / 插件类型，无需解析 UE C++ 宏；
- LSP 端很薄：Node + Peggy 做轻量解析，全部符号查询走内存数据库（前缀桶索引 + 继承缓存）；
- 同一 socket 复用：类型库、诊断、资产库、编辑器命令（跳转/开资产/建蓝图）。

**代价 / 限制**
- **强依赖编辑器运行**：UE 没开则没有引擎类型提示（仅剩本地脚本符号），重连需全量重传类型库；
- 类型库是快照式同步：引擎热重载 / 热编译后依赖重新推送（DebugServer 在脚本重编译后向订阅客户端重发数据库与诊断）；
- `.as` 之外不感知 C++ 源码，C++ 符号跳转只能交给编辑器端。

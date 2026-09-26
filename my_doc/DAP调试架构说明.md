# Unreal Angelscript DAP（调试适配器）架构说明

> 对应源码：
> - `extension/src/debug.ts` —— DAP 会话实现（`ASDebugSession`）
> - `extension/src/unreal-debugclient.ts` —— UE 调试客户端（TCP + 消息封装）
> - `extension/src/extension.ts` —— 调试器注册（`ASDebugConfigurationProvider`）
> - `extension/src/debugAdapter.ts` —— 独立调试适配器进程入口（`ASDebugSession.run()`）
> - 引擎侧：`AngelscriptCode/Private/Debugging/AngelscriptDebugServer.cpp`（及 Angelscript 引擎的 LineCallback 机制）

## 1. 总体架构

```
┌───────────────────────────────────────────────────────────────────────────┐
│  VSCode                                                                   │
│                                                                           │
│  ┌───────────────────┐  DAP (JSON-RPC over socket) ┌───────────────────┐  │
│  │ Debug UI          │ ◄─────────────────────────► │  ASDebugSession   │  │
│  │ (breakpoints /    │   local Net server started  │  (LoggingDebugSes │  │
│  │  stepping /       │   in extension.ts,          │   sion + unreal-  │  │
│  │  variables /      │   listening on random port  │   debugclient)    │  │
│  │  watch)           │                             │                   │  │
│  └───────────────────┘                             └─────────┬─────────┘  │
└──────────────────────────────────────────────────────────────┼────────────┘
                                                     TCP 127.0.0.1:27099
                                                     (same protocol as LSP,
                                                      separate connection)
                                                                │
┌───────────────────────────────────────────────────────────────┼────────────┐
│  Unreal Editor process                                        ▼            │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ AngelscriptDebugServer                                               │  │
│  │  - bIsDebugging / bIsPaused state machine                            │  │
│  │  - breakpoint tables (line / exception filters / data breakpoints,   │  │
│  │    max 4 hardware breakpoints)                                       │  │
│  │  - registers AngelScript engine LineCallback                         │  │
│  │    (per-line callback = stepping / breakpoint checks)                │  │
│  │  - PingAlive keepalive sent from Tick                                │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

要点：

- DAP 适配器**运行在扩展宿主进程里**（inline 模式），不是独立的 exe：首次启动调试时 `ASDebugConfigurationProvider.resolveDebugConfiguration` 起一个本地 `Net.createServer`（随机端口），把端口号塞进 `config.debugServer`，让 VSCode 以 server 模式连接（`extension.ts:445-472`）。`debugAdapter.ts` 的 `ASDebugSession.run()` 则保留为传统的独立进程入口（package.json 的 `program`/`runtime`）。
- 调试适配器与 UE 的通信**复用 LSP 同一套 TCP 二进制协议**（端口 27099，`[uint32 长度][uint8 类型][payload]`），但消息子集不同，且 `unreal-debugclient.ts` 是独立的一份实现（用 `EventEmitter` 分发消息，而非 LSP 的直接回调）。

## 2. 模块职责

| 文件 | 职责 |
|---|---|
| `debug.ts` | DAP 协议实现：initialize/launch/setBreakpoints/stackTrace/scopes/variables/evaluate/continue/pause/stepIn/Out/next、数据断点（`dataBreakpointInfo` / `setDataBreakpoints`）、异常过滤器、自定义请求 `angelscript/stopPIE` |
| `unreal-debugclient.ts` | UE socket 封装：`connect/disconnect`、全部 `send*` 消息构造器、`readMessages` 粘包处理、`EventEmitter` 事件分发（`unreal.events`，全局单例） |
| `extension.ts` | `ASDebugConfigurationProvider`：F5 无配置时自动生成 launch 配置、管理 DAP server 生命周期；`ASEvaluateableExpressionProvider`：hover 求值支持 |
| 引擎侧 `AngelscriptDebugServer.cpp` | 调试服务器：断点管理、LineCallback 执行、变量/求值序列化、`DebugServerVersion` 协商 |

版本协商（协议兼容的核心）：

- DAP 端 `debugAdapterVersion = 2`，在 `StartDebugging` 消息里带给引擎；
- 引擎回 `DebugServerVersion` 消息，DAP 存为 `debugServerVersion`；
- 之后多处按版本开关兼容行为：`debugServerVersion >= 2` 才读变量的 `address + valueSize`（数据断点依赖）、`> 0` 才读调用栈里的 `moduleName` 并做源码路径重定位。

## 3. DAP ↔ UE 消息映射表

| DAP 请求（VSCode → 适配器） | 适配器动作（→ UE） | UE 异步回推 | 适配器 → VSCode |
|---|---|---|---|
| `initialize` | `connect` + `RequestBreakFilters` | `BreakFilters` | 填充 `exceptionBreakpointFilters` 后发送 initialize response + `InitializedEvent` |
| `launch` | `connect` + `StartDebugging(version)`；先重放已缓存的断点（`ClearBreakpoints` + `SetBreakpoint`）；`await configurationDone` | `DebugServerVersion` | launch response |
| `setBreakpoints` | 缓存断点；若已连接则 `ClearBreakpoints(path, module)` + 每个断点 `SetBreakpoint(id, path, line, module)` | `SetBreakpoint`（实际绑定行号回告） | `BreakpointEvent('changed'/'removed')`（断点漂移/无效行） |
| `setExceptionBreakpoints` | `BreakOptions(filters)` | — | response |
| `threads` | —（无真实线程） | — | 固定单线程 `Thread(1, "Unreal Editor")` |
| `stackTrace` | `RequestCallStack` | `CallStack` | response（带排队机制，见 §4） |
| `scopes` | —（本地构造） | — | Variables / this / Globals 三个 Scope |
| `variables` | `RequestVariables(handle 解析出的表达式路径)` | `Variables` | response |
| `evaluate`（含 hover 求值） | `RequestEvaluate(expression, frameId)` | `Evaluate` | response |
| `continue` / `pause` | `Continue` / `Pause` | `HasContinued`（继续） | response / `ContinuedEvent` |
| `next` / `stepIn` / `stepOut` | `StepOver` / `StepIn` / `StepOut` | `HasStopped` | response |
| `disconnect` | `StopDebugging` + `Disconnect`，清空数据断点 | — | response |
| `dataBreakpointInfo` | —（查本地 `variableStore`） | — | dataId（仅 1/2/4/8 字节变量可用） |
| `setDataBreakpoints` | `SetDataBreakpoints`（id + 8 字节地址 + size + hitCount + 是否 C++ 断点） | `ClearDataBreakpoints`（引擎侧主动清除时） | response（超过 4 个标为 unverified） |
| 自定义 `angelscript/stopPIE` | `StopPIE` | — | response |

UE 主动推事件（无对应请求）：

| UE 消息 | 适配器动作 |
|---|---|
| `HasStopped`（reason/description/text） | `StoppedEvent`，reason == 'exception' 时带文本并存为 `previousException` 供 `exceptionInfo` 请求 |
| `HasContinued` | `ContinuedEvent` |
| socket 关闭/出错 | `TerminatedEvent`（结束调试会话） |

## 4. 关键实现细节

### 4.1 异步响应的“等待队列”模式

TCP 消息没有请求 ID，而 DAP 请求必须一一应答。适配器把 pending 的 response 挂到 FIFO 队列里（`waitingTraces` / `waitingVariableRequests` / `waitingEvaluateRequests`），UE 每回一条消息就取出队首 response 填充 body 并 `sendResponse`。因此**隐含假设：UE 严格按请求顺序串行回复**。

### 4.2 变量引用与表达式路径

- `scopes` 阶段为每个 frame 造三个虚拟引用：`frameId:%local%`（局部变量）、`frameId:%this%`、`frameId:%module%`（Globals）；
- `variablesRequest` 把 handle 字符串直接发给 UE，UE 返回变量列表后，用 `combineExpression`（父路径 + `.name` 或 `[index]`）为有成员的变量创建新的 handle——**变量树是按表达式路径惰性展开的**；
- 展开后的变量同时存入 `variableStore`（key = 表达式路径），供数据断点查询地址/大小；
- 名字以 `$` 结尾的变量被标记为 `method` 呈现（说明符/属性伪装成变量的场景）。

### 4.3 断点生命周期

```
VSCode setBreakpoints（用户改断点）
  → 适配器先本地缓存（断点可以在连接前设置）
  → 已连接：clearBreakpoints(文件) + setBreakpoint(id, path, line, module)
       （module 名 = 相对工作区根路径，"/" → "."，去掉 .as；与 LSP 的模块命名规则一致）
  ← UE 回 SetBreakpoint(文件, 实际行, id)：
       - 行号 != 请求行 → 断点"漂移"到有代码的行 → BreakpointEvent('changed', verified)
       - 行号 == -1     → 该行无代码 → BreakpointEvent('changed', unverified)
       - 与已有断点重叠 → 删除新断点 → BreakpointEvent('removed')
     （多个事件用 setTimeout 错开 1ms 发送，规避 VSCode 事件洪峰问题）
launch 时：把 launch 之前设置的所有断点按文件重放一遍
```

### 4.4 调用栈与源码映射

- 帧数据：函数名（去掉 `_Implementation` 后缀）、源路径、行号、模块名（v1+）；
- `::` 开头的路径是引擎内嵌代码/外部帧，显示为 `presentationHint: 'deemphasize'` 的 label 帧，不可跳转；
- 源路径重定位（`resolvePathsToSourcePath`）：UE 给的绝对路径在本机不存在时，用 `moduleName`（`Foo.Bar` → `Foo/Bar.as`）在工作区根目录列表中逐个探测存在性——支持远程/不同机器上的工程布局。

### 4.5 数据断点（v2 协议新增能力）

- 依赖引擎在 `Variables` 消息里附带每个变量的 64 位地址和值尺寸；
- 只支持 1/2/4/8 字节的标量，accessTypes 仅 `'write'`；
- 最多 4 个（硬件断点限制 `SUPPORTED_DATA_BREAKPOINT_COUNT`），超出的返回 unverified；
- 可配置触发命中次数（`hitCount`，-1 表示每次触发），且可选拆分为 C++ 层断点（`UnrealAngelscript.dataBreakpoints.cppBreakpoints.*` 设置）；
- 引擎侧也可能自行清除（如变量销毁），通过 `ClearDataBreakpoints` 回推同步 UI。

### 4.6 引擎侧执行模型（AngelscriptDebugServer.cpp）

- `StartDebugging` → `bIsDebugging = true`，并向 Angelscript 引擎注册/启用 **LineCallback**（脚本引擎每执行一行字节码回调一次）；
- LineCallback 中检查：是否命中行断点 / 是否 `bBreakNextScriptLine`（step over 实现）/ 是否有数据断点触发 → 命中则置 `bIsPaused` 并回 `HasStopped`，**暂停方式是阻塞在回调里**（UE 游戏线程停住）；
- `StepOver` = `bBreakNextScriptLine = true` + 取消暂停（同帧内下一行脚本再停）；
- `StopDebugging`/断开 → `bIsDebugging = false`、清空全部断点、更新 LineCallback 注册状态；
- Tick 中定期向调试客户端发 `PingAlive` 保活。

## 5. 一次典型调试会话时序

```
F5（launch）
  └─ resolveDebugConfiguration：起本地 DAP server（随机端口）
  └─ DAP: initialize ──► connect(UE) + RequestBreakFilters
       ◄── BreakFilters ◄── ...
       initialize response（含异常过滤器）+ InitializedEvent
  └─ DAP: setBreakpoints × N（断点写入 UE，等待 SetBreakpoint 回告校正行号）
  └─ DAP: setExceptionBreakpoints ──► BreakOptions
  └─ DAP: configurationDone
  └─ DAP: launch ──► StartDebugging(v2) ◄── DebugServerVersion ──►
       （PIE 中脚本执行到断点行）
       ◄── HasStopped("breakpoint") ──► StoppedEvent
  └─ DAP: threads / stackTrace ──► RequestCallStack ◄── CallStack ──►
  └─ DAP: scopes（本地）→ variables ──► RequestVariables("1:%local%") ◄── Variables ──►
  └─ DAP: evaluate（hover）──► RequestEvaluate ◄── Evaluate ──►
  └─ DAP: next/continue ──► StepOver/Continue ...（循环）
  └─ 停止调试：disconnect ──► StopDebugging + Disconnect ──► TerminatedEvent
```

## 6. 设计取舍

**优点**
- DAP 适配器极薄：无脚本执行引擎，纯协议翻译（DAP JSON ↔ 自定义二进制），真正的调试逻辑都在引擎侧；
- 与 LSP 复用同一个 UE 端口和消息格式（`MessageType` 枚举共享），LSP 连接断开不影响调试连接；
- 无请求 ID 的串行协议 + FIFO 等待队列，实现简单、消息体积极小；
- 数据断点直接利用变量真实内存地址，支持落到 C++ 层的硬件断点。

**限制**
- 单线程模型（`THREAD_ID = 1`）：UE 游戏线程阻塞式暂停，无并行脚本执行的概念；
- 无条件断点（hit condition / logpoint）、无 watch 逐帧刷新，evaluate 的表达式求值完全依赖引擎侧 Angelscript 解释器；
- 消息无序号/校验，依赖“严格按序回复”的隐式约定，协议演进靠 `debugAdapterVersion` / `debugServerVersion` 两个版本号做条件解析；
- 行断点精度是“源码行”级（依赖 AngelScript 字节码行号表），漂移逻辑在适配器里做了不少 UI 侧补偿。

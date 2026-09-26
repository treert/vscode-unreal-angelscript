# vscode-unreal-angelscript 项目说明

本仓库是 Unreal Angelscript 的 VSCode 扩展实现（Hazelight 开源版的 fork），核心是 Angelscript 语言的 **LSP**（语言服务）与 **DAP**（调试适配器）实现。

## 仓库结构

- `extension/` — VSCode 扩展（LSP client + DAP 调试适配器入口）
  - `src/debug.ts` — DAP 会话实现（`ASDebugSession`）
  - `src/unreal-debugclient.ts` — UE 调试客户端（TCP 消息封装）
  - `src/extension.ts` — 扩展入口、调试器注册、自定义请求
- `language-server/` — LSP 服务（Node.js，esbuild 打包）
  - `src/server.ts` — LSP 入口，连接 UE、解析队列驱动
  - `src/database.ts` — 内存符号数据库（UE 类型 + 脚本符号）
  - `src/as_parser.ts` — `.as` 模块解析（Peggy 语法）、作用域/类型解析
  - `src/parsed_completion.ts` — 补全、签名帮助
  - `src/unreal-buffers.ts` — LSP↔UE 二进制协议
  - `../pegjs/` — Angelscript 的 PEG 语法文件
- `my_doc/` — 架构分析文档（本仓库的分析产出）
  - `LSP架构说明.md` — LSP 总体架构、类型数据库、通信协议
  - `DAP调试架构说明.md` — DAP 适配器架构、消息映射
  - `引擎侧断点判定细节.md` — 引擎侧 DebugServer 断点/单步/数据断点机制
  - `AngelScript虚拟机架构说明.md` — VM 寄存器/数据栈/调用栈、ExecuteNext 解释器、caller 委托调用桥、异常展开、JIT 挂点
  - `Precompile与StaticJIT架构说明.md` — 字节码缓存（PrecompiledData）、C++ 离线转译流水线（AS_JITTED_CODE）、FJITDatabase 指针重定位、去虚化与 FloatingStack
  - `LSP语法解析流水线说明.md` — pegjs 多入口语法与容错设计、语句切分与 AST 缓存、四阶段队列、作用域/符号查找链、类型数据库混装 UE 类型
  - `类型绑定架构说明.md` — FAngelscriptType 类型操作接口、TypeFinder、caller thunk 自动生成、Binds.Cache 双数据源、UFunction 双路绑定（Callable/Event）

## 运行时（引擎侧）

as 脚本真正的运行时不在本仓库，位于：

- 引擎目录：`D:\WorkGit\UnrealEngine`（源码构建版）
- Angelscript 插件：`D:\WorkGit\UnrealEngine\Engine\Plugins\Angelscript`
  - `Source/AngelscriptCode/` — 插件主体（Manager、DebugServer、类型绑定、ClassGenerator）
  - `Source/Angelscript/ThirdParty/` — 修改过的 AngelScript 解释器源码（含调试钩子改造）

LSP/DAP 与运行时的通信：TCP `127.0.0.1:27099`，自定义二进制协议（`[uint32 len][uint8 type][payload]`），类型数据库、诊断、断点、变量求值都走这条通道。

## 技术分析方向（后续计划）

- [x] LSP 架构（补全如何获取 UE 引擎函数）
- [x] DAP 架构（断点/单步/数据断点的实现链路）
- [x] 引擎侧断点判定（LineCallback、调试寄存器）
- [x] AngelScript 虚拟机（字节码执行、Context/调用栈）
- [x] Precompile / StaticJIT（脚本预编译加速）
- [x] 语法解析（pegjs 语法、LSP 侧的作用域/类型解析流水线）
- [x] 类型绑定（UE 反射 ↔ AngelScript 类型的桥接）

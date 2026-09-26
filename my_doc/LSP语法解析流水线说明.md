# LSP 语法解析与类型解析流水线说明（pegjs 语法、作用域/类型解析）

> 对应源码（本仓库 `language-server/`）：
> - `pegjs/angelscript.pegjs`（37KB，~130 条规则）—— AngelScript 的 PEG 语法（Peggy），预编译为 `angelscript.js`（351KB parser）
> - `grammar/node_types.js` —— AST 节点类型枚举（与 C++ 侧 as_parser.cpp 的节点体系平行）
> - `src/as_parser.ts`（232KB）—— 语句切分、作用域树、符号/类型解析核心
> - `src/database.ts`（80KB）—— 内存符号数据库（DBType/DBMethod/DBNamespace + 查找链）
> - `src/server.ts` —— LSP 入口、四级解析队列、UE socket 连接
> - `src/parsed_completion.ts`（178KB）—— 补全/签名帮助（以解析流水线为基础）
>
> 前置阅读：《LSP架构说明》（总体架构与通信协议）。

## 0. 一句话总览

LSP 不复用引擎的 C++ 编译器——它用 **Peggy/PEG 语法独立实现了一套"容错优先"的解析器**：先把文件按词法切成语句（statement），每条语句单独走 PEG 解析（错了不影响别的语句），再构建作用域树、把脚本声明提升进内存符号数据库，最后在"标识符→符号→类型"的查找链上完成补全/悬停/诊断。UE 引擎类型通过 socket 以 JSON 形式灌入同一数据库，脚本类型与 C++ 类型在 `DBType` 层面无差别混合。

## 1. 总体数据流

```
+------------------ UE Editor (engine-side DebugServer, port 27099) ------------------+
|  DebugDatabase (JSON string stream, chunked) / DebugDatabaseFinished /             |
|  AssetDatabase / Diagnostics (real compiler errors) / DebugDatabaseSettings        |
+--------------------------------------+---------------------------------------------+
                                       |  server.ts:106-247 (connect_unreal, on data)
                                       v
                       typedb.AddTypesFromUnreal(dbObj)    <- C++ types enter DB
                       FinishTypesFromUnreal + AddPrimitiveTypes
                       ReResolveAllModules()               <- re-resolve all after
                                       |                      type DB is ready
+--------------------------- LSP (Node.js) ---------------+--------------------------+
|  .as files on disk -> ASModule -> 4-stage queues (server.ts:439-536)               |
|    LoadQueue(200/tick) -> ParseQueue(10/tick)                                       |
|      -> PostProcessTypesQueue(50/tick) -> ResolveQueue(20/tick)                     |
|                                       |                                             |
|  Per ASModule: rootscope (ASScope tree) + types / globalSymbols /                  |
|    semanticSymbols / moduleDependencies, all stored in ModuleDatabase               |
+-------------------------------------------------------------------------------------+
```

四级队列是**优先级串行**的（`else if` 链）：Load 没消费完不 Parse，Parse 没完不 PostProcess……每 tick 只跑一种阶段，批大小按单步开销调校（IO 200 / 解析 10 / 后处理 50 / 解析 20），`setTimeout(TickQueues, 1)` 让出事件循环，避免长阻塞。PostProcess/Resolve 阶段前有 `CanResolveModules()` 门控（server.ts:599）——**UE 类型库没传完不解析**（类型库超时 `UnrealTypesTimedOut` 后才放行，供无引擎时离线使用）。

## 2. Peggy 语法（angelscript.pegjs）

### 2.1 结构：多入口 + 四个启动规则

语法文件顶部用 JS 辅助函数构造 AST 节点（`Literal/Identifier/Compound/CompoundOperator` 等），规则产出统一的 `{type: node_types.X, children, value, operator, start, end}` 形状。四个入口规则对应四类作用域（as_parser.ts:6329-6354 按 scopetype 选择）：

| startRule | 使用场景 | 顶层规则 |
|---|---|---|
| `start` | 函数体/代码块/字面量资产 | `statement = if/return/else/switch/for/while/case/... var_decl / assignment / incomplete_var_decl` |
| `start_global` | 全局作用域 | `global_declaration = delegate/event/struct/class/enum/namespace/asset_decl / ufunction 函数签名 / var_decl / incomplete_var_decl` |
| `start_class` | 类作用域 | `access_decl / constructor / destructor / class_method_decl / class_property_decl` |
| `start_enum` | 枚举作用域 | `enum_statement` |

### 2.2 关键容错设计（LSP 与编译器的本质差异）

PEG 是**贪心正序**匹配（无回溯跨选择点），失败即抛异常——LSP 要面对的是"写了一半的代码"。语法里的对策：

- **`incomplete_var_decl`**（:408）：`var_decl` 的残缺版本——正在输入的声明（没写完分号/初始化器）也能解析出一个部分 AST，`ParseStatement` 还把 `endsWithSemicolon`、`precedesBlock` 作为 parser 选项传入（:6360-6364）辅助判定；
- **语句级失败隔离**：单条语句解析失败只置 `statement.parseError`，同文件其它语句照常（见 §3）；
- **`_` 与 `__` 空白规则内联注释**（:97-105）：`//`、`/* */`、`#` 预处理行全部在词法层吸收，AST 里没有注释节点（文档注释 `comment_documentation` 例外，挂到声明上）；
- **操作符前瞻消歧**：`op_binary_sum = @"+" !("=" / "+")` 之类，用负向前瞻把 `+`/`+=`/`++` 在 token 层分开，`op_compound_assignment` 列全 8 种复合赋值。

### 2.3 与引擎语法的平行关系

`node_types.js` 的节点枚举与引擎侧 `as_scriptnode.h` 概念对齐（ClassDefinition/FunctionCall/MemberAccess/BinaryOperation...），但**这是两套独立实现**：引擎语法（as_parser.cpp 的手写递归下降）服务于"必须全对才编译"，本语法服务于"部分对也要给补全"。因此规则集更松：接受 `ufunction_macro`（UFUNCTION(...)）、`asset_decl`（字面量资产）等 UE 扩展语法。

## 3. 语句切分：为什么不是整文件一把解析

`ParseModule`（as_parser.ts:784）的第一步不是调 Peggy，而是 **`ParseScopeIntoStatements`（:5677）手写扫描**：追踪大括号/圆括号深度、字符串（`" '`）、转义、行注释/块注释/预处理器状态机，把 scope 文本切成 `ASStatement{content, start_offset, end_offset, rawIndex}` 序列（分号或 `{` 结束一条）。然后 `ParseAllStatements`（:6029）逐条调 `ParseStatement` → `PEGGY_GRAMMAR.parse(statement.content, {startRule...})`。

这个"先切后解析"的决定性好处：

1. **增量**：`UpdateModuleFromContent`（:977）diff 新旧内容算出 `lastEditStart/End`；配合 **`GetCachedStatementParse`**——语句文本没变就直接复用上次的 AST（模块统计 `loadedFromCacheCount` vs `parsedStatementCount`），编辑一个函数只重解析受影响的几条语句；
2. **容错**：坏语句不拖垮全文件；
3. **编辑中分裂**（:6052-6100）：正在编辑的语句解析失败时，`SplitStatementBasedOnEdit` 按编辑点把语句一分为二再各自解析——覆盖"新语句打到旧语句前面、分号还没敲"的最常见打字状态；多行 `var_decl` 还会在编辑行之前尝试不分裂解析。

scope 树的构建也在这一层：`DetermineScopeType`（:6032 在 ParseAllStatements 开头）根据**前一条语句的 AST**决定 `{}` 块的身份——前一条是 `ClassDefinition` → 这块是 Class scope；是函数签名 → Function scope；`for_statement` 直接携带自己的子 Code scope。scope 与语句通过双向链表（`ASElement.previous/next`）串接，`getScopeAt/getStatementAt` 都是 offset 区间上的线性/递归查找。

## 4. 三遍流水线：Parse → PostProcessTypes → Resolve

### 4.1 Parse（ParseModule）：声明提升

`GenerateTypeInformation(scope)`（:1826）遍历 scope 树，把声明**提升进符号数据库**：

- 前一语句是 `ClassDefinition` → `AddDBType` 建 `DBType`（supertype 默认 `UObject`），macro（UCLASS(...)）解析进 `macroSpecifiers/macroMeta`，`scriptkeywords` 元数据变成类型 keywords（供补全）；scope 与 DBType 互挂（`scope.dbtype`）；
- `StructDefinition` → isStruct 的 DBType；`NamespaceDefinition` → `DeclareNamespace`（支持 `A::B::C` 逐级建）；enum/delegate/event/函数/全局变量同理；
- **变量在函数体内提升**：`scope.variables` + `variablesByName`（Map，含参数 `isArgument`、`auto` 变量的 `node_expression` 留待推导）。

这一遍之后模块"有骨架无语义"——`module.types/globalSymbols` 就位，但表达式里的符号还没解析。

### 4.2 PostProcessTypes：生成代码处理

`ProcessScriptTypeGeneratedCode`（generated_code.ts）处理脚本类的 UE 侧生成代码语义（蓝图书签、默认组件等衍生符号），为 Resolve 提供完整成员集。

### 4.3 Resolve（ResolveModule，:837）：不动定点迭代

```
do {
    module.resolved = true;
    ResolveAutos(rootscope);          // auto 变量按初始化表达式推导类型（含迭代器解包）
    DetectScopeSymbols(rootscope);    // 核心：全量符号解析（见 §5）
    foreach (dependency in moduleDependencies):
        未 parsed → LoadAndParseModule(dep); module.resolved = false
        未 resolved → EnsureTypeHierarchyFullyParsed(dep 的每个类); 可能再置 false
        未 typesPostProcessed → PostProcessModuleTypes(dep); module.resolved = false
} while (!module.resolved)            // 发现新依赖就整体重跑
```

**依赖是解析中动态发现的**：`DetectIdentifierSymbols` 每解析到一个跨模块符号就 `markDependencySymbol/markDependencyType`（:4888/4904），`moduleDependencies` 随之增长。不动点循环保证：用到别模块的类，那个模块（及其父类链）必定先解析完，成员查找才不会漏。

**父类链保障 `EnsureTypeHierarchyFullyParsed`（:907）**：沿 supertype 链上溯，任何一层类型查不到就去 **`PreParsedIdentifiersInModules`**——启动时对全部 `.as` 文件做过一遍**标识符预扫描**（类名→模块集合），按名字找到"可能声明这个类的模块"直接解析，再查一次；查不到就放弃（保持容错）。这是用空间（预扫描索引）换启动时序解耦。

Resolve 完成后触发 `resolveCallbacks`（悬停/语义高亮等请求在此等待）、清 `cachedStatements`，诊断刷新（`UpdateScriptModuleDiagnostics`）。`ReResolveAllModules`（UE 类型库到达/断线重连后）把所有模块的 resolved 置脏重排。

## 5. 符号/类型解析：查找链

### 5.1 ResolveTypeFromExpression（:2784）：表达式 → 类型

按 AST 节点类型分派，典型递归：

| 节点 | 推导 |
|---|---|
| `Identifier X` | `ResolveTypeFromIdentifier`（走 §5.2 查找链） |
| 字面量 | `1`→int、`0.f`→float（floatIsFloat64 设置下→float32）、`"s"`→FString、`n"x"`→FName、`nullptr`→UObject |
| `X.Y` | 递归左类型 → `ResolvePropertyType(左类型, Y)` |
| `X::Y` | 命名空间访问；`Super::Y` 特判为父类作用域 |
| `X(...)` 函数调用 | 先 `ResolveFunctionFromExpression`；失败再试"同名类型构造"；成功则返回类型——**`determinesOutputTypeArgumentIndex`** 处理 `TArray<T>::Get()` 这类由实参决定返回的模板方法（:2913-2920） |
| `X[]` / `X op Y` / `-X` / `X++` | `ResolveTypeFromOperator` 查运算符重载方法（opIndex/opAdd/...） |

### 5.2 DetectIdentifierSymbols（:4862）：标识符的完整查找链

语义解析的原子操作（ DetectScopeSymbols → DetectNodeSymbols 对每条语句 AST 递归调用），查找顺序：

```
1. 局部变量：scope 链上溯（仅 isInFunctionBody 的 scope），variablesByName 命中
   → 记录 ASSemanticSymbol（局部变量/参数、读写访问）→ LookupType(变量 typename)
2. 当前类成员：scope.getParentType().findFirstSymbol(name)
   → 成员变量/成员函数符号 + markDependencySymbol
3. 属性访问器：Get<Name>/Set<Name>（引擎无属性语法，LSP 模拟 AS_PROPERTY_ACCESSOR_MODE）
4. 全局符号：LookupGlobalSymbol(命名空间链)
   —— mixin 函数有可见性过滤（类内仅当 args[0] 是本类基类时可见，:4936-4946）
5. 命名空间/类型本身（identifier 也可以是类型名，如静态调用）
```

`ASParseContext`（:3790）携带跨节点状态：`isWriteAccess`（赋值左侧）、`isRootIdentifier`（unused 警告豁免）、`argumentFunction`（回调实参上下文）、`isResolvingFunction`——成员访问左值会把 writeAccess 向下传一层（值类型局部变量 hack，:3884-3900）。

### 5.3 查找的终点：database.ts

- **`LookupType(namespace, typename)`（:2202）**：先查无命名空间的 `TypesByName`（命中还要校验命名空间兼容）；带 `::` 的逐级下钻命名空间、每级失败回退到父命名空间再试（namespace 链上溯）；最后是**模板实例化**——`TArray<FVector>` 按正则拆出基类型与子类型，`createTemplateInstance` 克隆出符号集替换后的新 DBType（:2256-2276）；
- **DBType**（:628）：脚本类型与 UE 类型同构。`symbols: Map<name, DBSymbol|DBSymbol[]>`（重载是数组）、`symbolsByPrefix`（前缀索引供补全）、`macroSpecifiers`（UCLASS/UPROPERTY 元数据）、`moduleOffset*`（声明位置，供跳转）；
- **`inheritsFrom/findFirstSymbol`** 沿 supertype 链查父类成员——链上的 `LookupType` 每次都是命名空间感知的；
- **UE 类型入口**（server.ts:161-183）：`MessageType.DebugDatabase` 消息携带 JSON（引擎侧序列化的反射类型，含 properties/methods/doc/keywords/subtypes/inherits），`DBType::fromJSON`（:713）直接建库；分片到达由 1 秒超时 `DetectUnrealTypeListTimeout` 判定结束，`DebugDatabaseFinished` 后 `AddPrimitiveTypes + ReResolveAllModules`。

### 5.4 悬停/补全如何复用同一套流水线

所有语言特性都不自带解析，而是"等 Resolve 完成再查"：`WaitForResolveSymbols`（server.ts:952-989）轮询 `module.resolved`（20 次 × 间隔重试），`GetParsedCompletions` 直接在已解析的 scope/DBType 上工作。补全主入口 `Complete`（parsed_completion.ts:194-303）的上下文推导：

```
GenerateCompletionContext(offset-1)  ← 光标前文本的词法/语法状态
  ├─ isIgnoredCode（注释/字符串内）→ 无补全
  ├─ isNamingSomethingNew → 只补类型名/命名名
  ├─ AddCompletionsFromUnrealMacro/AccessSpecifiers → UFUNCTION(...) 括号内的说明符
  └─ priorType != null（"X." / "X::" 之后）→ 只在该类型上补
       否则 → 搜索域 = 所在类 + 命名空间链（逐级） + 局部变量（scope 链）
              + this/Super 关键字 + mixin + 关键字 + 调用实参名 + override 片段
```

`Signature`（:569）用 `GetDetermineTypeFromArguments` 反查实参类型给重载排序；`GetExpectedTypeAtOffset`（:4929）供内联提示/代码动作取期望类型。模块依赖规则（`isValidModuleDependency`/`resolveModuleIsolation`，:3799-3835）在同一文件里实现模块隔离检查（bDisableModuleIsolation 设置）。

## 6. 诊断的双轨制

- **`Diagnostics` 消息**（引擎侧 asCCompiler 真实编译错误，server.ts:110-160）：按文件整组替换，`UpdateCompileDiagnostics`——**权威报错**；
- **LSP 本地诊断**（ls_diagnostics.ts，Resolve 完成时触发）：基于自研解析器的宽松检查（unused 变量、未知符号等），只报引擎不会抓的问题。两轨并存是因为本地解析器容错太强，报不了类型错误；而引擎编译有热重载延迟。isInfo 级的引擎诊断只在本地已有同行诊断时才显示（:131-142），避免"行号对不上的整行波浪线"噪音。

## 7. 值得注意的设计总结

| 设计 | 动机 |
|---|---|
| 手写词法切语句 + 逐语句 PEG 解析 | 增量（编辑只重解析受影响语句 + AST 缓存）与容错（坏语句局部化）是 LSP 第一需求 |
| 多 startRule（global/class/enum/code） | 同一语法服务不同作用域，避免"全文件语法"的超集复杂度 |
| incomplete_var_decl / 语句分裂 | 覆盖"正在输入"状态，这是与编译器语法的最大分叉点 |
| 声明提升（Parse）与语义解析（Resolve）分离 + 不动点迭代 | 跨模块依赖在解析中动态发现，迭代保证查找时依赖链必就绪 |
| PreParsedIdentifiersInModules 预扫描 | 类名→模块索引，绕开"父类在哪个文件"的解析时序问题 |
| 局部→类成员→访问器→全局→命名空间 的显式查找链 | 与 AngelScript 语言作用域规则一致；每级命中都产出 semantic symbol（高亮/rename/引用共用） |
| UE 类型与脚本类型统一进 DBType | 补全/悬停/类型链在两类符号上行为一致；模板实例化对两者同样生效 |
| 类型库未就绪不 Resolve（CanResolveModules） | 避免"先用空库解析再全量返工"的浪费；离线超时兜底 |
| 语言特性请求一律"等 resolved"（轮询 resolveCallbacks） | 单一解析真值来源，所有特性共享同一次解析结果 |

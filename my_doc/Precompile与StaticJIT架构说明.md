# Precompile / StaticJIT 架构说明（脚本预编译与转译加速）

> 对应源码（相对 `Engine/Plugins/Angelscript/Source/AngelscriptCode/`）：
> - `Private/StaticJIT/AngelscriptStaticJIT.h / .cpp` —— C++ 代码生成器主体（`FAngelscriptStaticJIT`、`FStaticJITContext`、输出文件组织）
> - `Private/StaticJIT/AngelscriptBytecodes.h / .cpp` —— 逐指令的"字节码 → C++"翻译表
> - `Private/StaticJIT/PrecompiledData.h / .cpp` —— 字节码/元数据二进制缓存（序列化 + 运行时重建 + 引用解析）
> - `Private/StaticJIT/StaticJITHeader.h / .cpp` —— 编译进生成代码的运行时支撑（注册、异常、系统调用桥）
> - `Public/StaticJIT/StaticJITConfig.h` —— 开关宏（`AS_CAN_GENERATE_JIT` / `AS_SKIP_JITTED_CODE` 等）
> - `Private/AngelscriptManager.cpp:340-530` —— 流程编排（命令行开关、缓存加载、GUID 校验）
>
> 前置阅读：《AngelScript虚拟机架构说明》§10（VM 侧 JIT 挂点）。

## 0. 一句话总览

**"StaticJIT" 不是 JIT**——它是**离线转译器（transpiler）+ 字节码缓存**：把 AngelScript 字节码翻译成等价 C++ 源码，随游戏一起用 MSVC/GCC 编译进二进制；运行时通过"函数 ID → 编译产物指针"的数据库直接调用。配套的 PrecompiledData 把解析/编译产物（类、函数、字节码、元数据）序列化成 `PrecompiledScript.Cache`，打包版启动时跳过整个 Preprocessor/编译器前端。

## 1. 三种运行形态与开关（AngelscriptManager.cpp:360-363）

| 形态 | 条件 | 脚本执行方式 |
|---|---|---|
| **编辑器/开发模式** | `GIsEditor` 或 `-as-development-mode` | 每次从 `.as` 源码完整编译，走解释器；支持热重载、断点调试 |
| **生成模式** | `-as-generate-precompiled-data` | 完整编译一次，产出 `AS_JITTED_CODE/*.cpp` + `PrecompiledScript.Cache` 后强制退出（`:496-510`） |
| **打包运行** | 非编辑器、非 commandlet、无 dev 标记 | 加载 `.Cache` 重建模块；有匹配的编译产物则全程调 C++ 转译函数 |

其他开关：`-as-ignore-precompiled-data`（忽略缓存）、缓存按配置分文件（`PrecompiledScript_Shipping/_Test/_Development.Cache`，`:441-450`）。`StaticJITConfig.h`：`AS_CAN_GENERATE_JIT`（Win/Linux）；**`WITH_EDITOR` 下强制 `AS_SKIP_JITTED_CODE`**——编辑器构建永不使用转译代码，生成代码文件里的全部内容被宏隔离。

## 2. 完整流水线

```
┌─ 生成阶段（编辑器/命令行带 -as-generate-precompiled-data）─────────────┐
│  创建 PrecompiledData + StaticJIT(bGenerateOutputCode=true)             │
│  SetJITCompiler(StaticJIT)；asEP_BUILD_WITHOUT_LINE_CUES=1（连SUSPEND都不生成）│
│  InitialCompile → asCModule::Build → 逐函数 JITCompile()                │
│      └─ FAngelscriptStaticJIT::CompileFunction：不编译，只登记           │
│         FunctionsToGenerate，*OutJITFunction=nullptr（本次仍走解释器）    │
│  WriteOutputCode() → <引擎根>/AS_JITTED_CODE/*.cpp（§4）                │
│  PrecompiledData->InitFromActiveScript() → Save(PrecompiledScript.Cache)│
│  RequestExit —— 一次运行同时产出两份产物                                  │
└──────────────────────────────────────────────────────────────────────┘
                              ↓ 构建系统把 .jit.cpp 编进游戏二进制
┌─ 运行阶段（打包版启动）────────────────────────────────────────────────┐
│  FStaticJITCompiledInfo JitInfo(GUID)  ← 生成代码静态初始化时注册        │
│  FStaticJITFunction::Register(Id, VMEntry, ParmsEntry, Raw) → FJITDatabase│
│  FJitRef_Function/Type/GlobalVar/PropertyOffset::Register → 查找表      │
│  Load(PrecompiledScript.Cache) → BuildIdentifier/GUID 校验（§5.1）      │
│  InitialCompile：模块不走 Preprocessor，从缓存重建（ApplyToModule×3）     │
│      └─ 重建 asCScriptFunction 时查 FJITDatabase.Functions[Id]          │
│         → jitFunction 三件套就位（PrecompiledData.cpp:533-540）          │
│         → JIT 函数连 bytecode 都不加载                                   │
│  解析查找表：旧指针值(key) → 新运行时指针，写回生成代码里的静态对象（§5.3） │
│  ClearUnneededRuntimeData() → 删除 PrecompiledData、清空 FJITDatabase   │
└──────────────────────────────────────────────────────────────────────┘
```

之后所有脚本调用（`Execute`、`asBC_CALL`、UE 反射 `MakeRawJITCall_*`）命中 `jitFunction` 直接执行 C++，与《虚拟机架构说明》§10 描述的挂点衔接。

## 3. PrecompiledData：可序列化的编译产物镜像

`FAngelscriptPrecompiledData`（PrecompiledData.h:612）是 asCModule 内部结构的一面"可序列化镜像"：

- **模块层**：`FAngelscriptPrecompiledModule{Functions, Classes, Enums, GlobalVariables, StaticsClassName, DeclaredEvents/Delegates, PostInitFunctions}`——类/函数/枚举/全局变量的完整签名 + UFunction 元数据（BlueprintCallable/Net/Exec 等 18 个标志位）+ **字节码本身**（`ByteCode` + `ByteCodeReferences`，引用表另存）。
- **指针问题的解法——"Reference"体系**：AngelScript 编译产物里到处是裸指针（类型、函数、全局变量地址）。序列化时全部替换成 `FAngelscriptPrecompiledReference{int64 OldReference}`——**生成那次运行时的指针数值本身**，配合按名字索引的 `TypeReferences / FunctionReferences / GlobalReferences / PropertyReferences` 表（名字+模块+命名空间+签名）。加载重建对象后，用旧指针做 key 查表得到新对象（`GetTypeInfo/GetFunction/GetGlobalVariable`）。这正是 FJitRef 查找表复用的同一套机制。
- **重建三阶段**（`ApplyToModule_Stage1/2/3`）：先建类型/签名，再建函数与字节码（引用翻译），最后全局变量初始化——对应 AngelScript 编译器的正常 Build 顺序。
- **跳过 Preprocessor**：`GetModulesToCompile()` 直接从缓存还原 `FAngelscriptModuleDesc`（编辑器版由 `FAngelscriptPreprocessor` 扫描源码生成）——**打包版连源码文件都不需要存在**。
- **内存策略**：`bMinimizeMemoryUsage` + `FMemMark`/`TPrecompiledAllocator`（memstack 分配器，整块释放）；`ClearUnneededRuntimeData` 在重建完成后丢弃诊断信息。JIT 函数不加载 bytecode，`LineNumbers` 等仅调试需要的字段可裁。
- **构建指纹**：`BuildIdentifier`（构建配置标识，Debug/Dev/Test/Shipping 四值）+ `DataGuid`（**每次构造 `FAngelscriptPrecompiledData` 时 `FGuid::NewGuid()` 生成的随机 GUID**，PrecompiledData.cpp:2583-2588——与脚本内容无关，仅作为"一对产物"的配对 token，写进生成代码的 `AngelscriptJitInfo.jit.cpp`）双校验。

## 4. 代码生成器：FAngelscriptStaticJIT

### 4.1 输出文件组织（WriteOutputCode，AngelscriptStaticJIT.cpp:3701-3960）

| 产物 | 内容 |
|---|---|
| `SharedHeaders/*.h` | 脚本类布局的 C++ 定义（`DetectScriptType` 生成），多模块共用；仅被 1 个模块用时直接内联进该模块文件 |
| 每模块一个 `.cpp` | 模块内全部函数的转译体 + `extern` 前向声明（跨模块函数引用）|
| `AngelscriptJitCode_N.jit.cpp` | **unity build 合并文件**，约 20 万行/个，`#include` 一批模块 cpp |
| `AngelscriptJitInfo.jit.cpp` | `AS_FORCE_LINK static const FStaticJITCompiledInfo JitInfo(FGuid(...))` —— 编译产物的 GUID 戳 |

细节：文件内容与磁盘比对，**没变就不重写**（增量迭代）；每文件头按当前构建配置生成 `#if UE_BUILD_XXX` 隔离；`AS_FORCE_LINK` 防止链接器裁掉看似无人引用的静态注册对象。生成目录在引擎根 `AS_JITTED_CODE/`，由**游戏工程的构建脚本**（不在本插件内）把 `.jit.cpp` 编进目标——插件侧只负责产出与运行时注册。

### 4.2 逐指令翻译：AngelscriptBytecodes.cpp

每条 VM 指令一个 `FAngelscriptBytecode_*` 实现，静态自注册进 `TMap<asEBCInstr, FAngelscriptBytecode*>`（`IMPL_BYTECODE_BEGIN/END` 宏，:55-56）。`Implement(FStaticJITContext&)` 输出 C++ 行。典型对照：

```cpp
// asBC_ADDi（:5141）→ 一行 C++：
Context.Line("{0} = {1} + {2};", VArg_SignedVar(0), VArg_SignedVar(1), VArg_SignedVar(2));

// asBC_SUSPEND（:3912）→ 空操作（return true 不输出任何行）——JIT 代码没有"行"概念
// asBC_CALL（:2646）→ FCallScriptFunction(ScriptFunction, IdArg, ResolveVirtual=false)
//                        生成通过 FJitRef_Function 运行期取函数指针的调用
```

### 4.3 FStaticJITContext：从"栈机器"到"寄存器机"的关键——FloatingStack

生成的函数开头（`GenerateNewFunction`:341+）模拟 VM 环境：

```cpp
SCRIPT_DEBUG_CALLSTACK_FRAME(...);        // AS_JIT_DEBUG_CALLSTACKS 时的调用栈帧 RAII
SCRIPT_ASSUME_NO_EXCEPTION();
alignas(8) asBYTE l_stack[N];             // VM 数据栈的 C++ 替身（仅必须物化时用）
asQWORD l_valueRegister; asBYTE l_byteRegister; float l_floatRegister; ...
void* l_objectRegister;
// 局部变量：LocalVariables 把 VM 栈偏移映射为具名 C++ 局部（FVariableAsLocal）
```

但**求值栈默认不落到 `l_stack`**：`FloatingStackExpressions`（AngelscriptStaticJIT.h:221-233）把每条 push 记成**符号表达式**（`FStackExpression{DWords, Expression, OffsetOnStack, bIsVolatile...}`），后续指令消费栈顶时直接内联表达式文本，直到真正需要内存地址/跨标签存活/函数调用时才 `MaterializeStackAtOffset`。效果：`(a+b)*c` 这类表达式链在生成代码里就是一行嵌套 C++ 表达式，**留给 MSVC 做常量折叠与寄存器分配**——这是"转译后接近原生性能"的最大来源。

两种压栈的代码级对照（AngelscriptStaticJIT.cpp:2437-2459）：

```cpp
// Push：立即物化 —— 真实内存写
Line("value_assign_safe<{0}>(&l_stack[{1}], {2});", StackType, Offset*4, Expression);

// PushVolatile：只记账 —— 不写内存，(偏移, 表达式) 进 FloatingStackExpressions
Expr.Expression = Expression;   /* 仅登记 */
```

`MaterializeStackAtOffset(Offset)`（:2539）在需要时才把某槽的表达式落地成一行赋值并从表移除。以 `x = (a+b)*c`（字节码 `PshV4 a; PshV4 b; ADDi; PshV4 c; MULi; StoreV4 x`）为例：

```cpp
// 无 FloatingStack 的朴素转译（每步写 l_stack，5 次内存往返）:
value_assign_safe<int>(&l_stack[0], a);
value_assign_safe<int>(&l_stack[4], b);
value_assign_safe<int>(&l_stack[0], *(int*)&l_stack[0] + *(int*)&l_stack[4]);
value_assign_safe<int>(&l_stack[4], c);
value_assign_safe<int>(&l_stack[0], *(int*)&l_stack[0] * *(int*)&l_stack[4]);
x = *(int*)&l_stack[0];

// FloatingStack 实际产出（PshV4 走 volatile，运算内联表达式，StoreV4 直接落变量）:
x = (a + b) * c;
```

注意与 VM 解释器的对照：同一个 `ADDi`，解释器执行时是真实的栈内存读写（`l_fp` 偏移访存）；转译代码里 `l_stack[]` 虽然存在但大部分时间不被写——转译器不需要自己实现优化，只负责"不挡路"，把优化留给 C++ 编译器。

配套状态跟踪同样是为了少物化、少分支：

- `EValueRegisterState` + 跳转汇合处的多状态检测（`bHasMultipleValueRegisterStates` 冲突时物化）；
- `bGuaranteeNonNull` / `bTopOfStackNonNull`：编译期能证明非空的指针，省掉空检查；
- `PendingNumericComparison`：比较指令延迟到跳转指令再合并成单个 C++ 比较；
- `FComputedOffsets`（:452-477）：**属性偏移能否硬编码**。脚本类/C++ 类布局可能在生成与运行之间变化的（UE 反射类型、TOptional 等模板），生成 `FJitRef_PropertyOffset` 运行期解析；确定不变的（FVector/FRotator 等白名单、原始子类型的模板）直接写死常量。`AS_JIT_VERIFY_PROPERTY_OFFSETS`（非 Shipping/Test）额外生成校验对象，运行期 `ensure` 硬编码值与真实偏移一致。

### 4.4 异常路径的生成

JIT 代码没有解释器的逐指令检查点，异常按两条路径生成：

- **显式抛出点**（空指针、除零、越界…）：调用 `FStaticJITFunction::SetNullPointerException(Execution)` 等（StaticJITHeader.cpp:119-159）→ 置 `Execution.bExceptionThrown = true` + `HandleExceptionFromJIT`（日志/调试服务器），然后**直接 return 逐层快速退栈**；
- **栈上存活对象的清理**：`ExceptionCleanupLabels`（:397-407）为每个需要清理的位置生成带标签的清理块（对应解释器的 `CleanStackFrame`/`DetermineLiveObjects`），异常退出时按当前位置跳到对应清理标签，释放已构造的临时对象后再退栈。

### 4.5 跨函数/系统函数调用

- **脚本调脚本**：默认经 `FJitRef_Function`（生成代码里的全局对象，`Get()` 取运行期解析后的函数指针）——因为热重载/多模块下函数地址不能写死。**去虚化**（`DevirtualizeFunction`:3646-3671）：非虚函数直接引用；虚函数但整个脚本集合中**无任何 override**（`FunctionsWithVirtualOverrides`，由 `AnalyzeScriptFunction` 预扫描得出）也直连目标；导入函数绑定到生成时的目标。去虚化后是纯 C++ 直接调用，可被内联。
- **脚本调 C++（系统函数）**：转译成 `FStaticJITFunction::ScriptCallNative(Execution, Function, l_sp, &l_valueRegister, &l_objectRegister)`（StaticJITHeader.cpp:166-257）——与 `asCContext::CallFunctionCaller` **逐行同构**的 caller 委托调用桥（this 指针校验、元数据首参、参数指针打包、`tld->activeFunction` 设置全一致）。generic 函数走 `ScriptCallGeneric`（`asCGeneric`）。`FJitRef_SystemFunctionPointer` 持有 `asCScriptFunction*` 的运行期解析结果。
- **三形态入口**：每个函数生成 `VMEntry`（VM 栈帧 `l_fp` 入口，供 `Execute`/`asBC_CALL` 调）、`ParmsEntry`（UE Parms 结构入口，供反射直调）、`Raw`（无参形态），即《虚拟机架构说明》§10 的 `jitFunction` 三件套。

### 4.6 生成模式下的注册产物

每个生成文件尾部 `StaticInitializationCode`（:3859-3882）：

```cpp
AS_FORCE_LINK static FStaticJITRegistration REGISTER_ModuleName = []() {
    FStaticJITFunction::Register(FunctionId_1234, &Func_1234_VMEntry, &Func_1234_ParmsEntry, &Func_1234_Raw);
    ...
    return FStaticJITRegistration{};
}();
```

进程启动（静态初始化）时这些 lambda 把全部函数指针灌入 `FJITDatabase::Functions`，同时 `FJitRef_*::Register` 把各查找槽位登记进对应数组（StaticJITHeader.cpp:32-84）。

## 5. 运行时：FJITDatabase 与指针重定位

### 5.1 一致性校验（AngelscriptManager.cpp:466-477）

```
FStaticJITCompiledInfo::Get()->PrecompiledDataGuid != PrecompiledData->DataGuid
  → "Loaded angelscript precompiled data does not match the transpiled C++
     in the game binary. Transpiled code will not be used!"
  → FJITDatabase::Clear()   ← 全部回退解释器，不崩、不半信半疑
```

缓存与转译代码**必须成对再生成**（GUID 随每次生成变化）；`IsValidForCurrentBuild` 的 `BuildIdentifier` 另挡构建配置错配——注意它**只比对 Debug/Dev/Test/Shipping 四个值**（PrecompiledData.cpp:2594-2612），不检测源码变化。

**重点：`DataGuid` 是随机数，不是内容哈希**（PrecompiledData.cpp:2583-2588）：

```cpp
FAngelscriptPrecompiledData::FAngelscriptPrecompiledData(asIScriptEngine* InEngine)
{
    DataGuid = FGuid::NewGuid();   // 构造即随机，与脚本内容零关联
}
```

因此**改 1 行和改 100 个文件效果相同**：重新生成的 Cache 其 GUID 必然不同于旧二进制里的 `JitInfo` GUID——转译代码全部失效、整库回退解释器，无任何 per-module/per-function 部分匹配。

### 5.1.1 GUID 不匹配的完整后果链（代码级验证）

只换 Cache 下发（新 Cache + 旧二进制）时的逐行执行路径：

```
Manager 启动：Load(新Cache) → IsValidForCurrentBuild ✓（配置没变）
  → CompiledInfo->PrecompiledDataGuid(旧二进制内嵌) != PrecompiledData->DataGuid(新随机)
  → Warning + FJITDatabase::Clear()                     (Manager.cpp:473-477)
      Clear() 清空全部 6 张表：Functions / FunctionLookups /
      SystemFunctionPointerLookups / GlobalVarLookups /
      TypeInfoLookups / PropertyOffsetLookups            (AngelscriptStaticJIT.cpp:19-34)
  → InitialCompile：仍从新 Cache 重建模块（不读源码、不重编译）
      每个函数：JITDatabase.Functions.Find(Id) → miss     (PrecompiledData.cpp:533)
      → jitFunction 三件套保持 null
  → PrepareToFinalizePrecompiledModules()：查找表已空，
    FJitRef_* 全局对象里的陈旧指针不被解析——但也永远不会被解引用
                                                          (PrecompiledData.cpp:2336-2394)
  → bStaticJITTranspiledCodeLoaded = Functions.Num() > 0 = false
                                                          (Manager.cpp:515)
```

关键细节（PrecompiledData.cpp:545-561）：

```cpp
// 生成函数特例：正常情况下假设"库非空时自己肯定有 JIT"而跳过 bytecode；
// Clear 后 Functions.IsEmpty() 为真 → 特例不成立 → 同样走 fallback
if (!bHasJITFunctions && Function->traits.GetTrait(asTRAIT_GENERATED_FUNCTION)
    && !JITDatabase.Functions.IsEmpty())
    bHasJITFunctions = true;

if (!bHasJITFunctions)
{
    Function->AllocateScriptFunctionData();       // ← 反而加载 bytecode
    scriptData->byteCode.SetLength(ByteCode.Num());
    FMemory::Memcpy(...);                          // Cache 里的字节码照常灌入
}
```

即：**回退是"解释执行缓存字节码"，不是"重新从源码编译"**——打包版仍不需要 `.as` 源码，功能完全正确，只是失去转译性能。之后 `Execute()`/`asBC_CALL`/`MakeRawJITCall_*` 全部命中 null 的 `jitFunction` 走 `ExecuteNext()`。

**为什么不做内容哈希/部分匹配**：函数 ID 由 `CreateFunctionId` 顺序分配（:2694-2699，冲突时递增），在中间插入/删除函数会使后续 ID 错位——两个函数内容都没变也可能 ID 对不上，而 ID 错位是"调错函数"级灾难。随机 GUID + 全对全错是最保守但绝对安全的方案；且转译代码静态链入二进制，运行时本无"替换单个函数"的机制，做半套匹配没有意义。

### 5.2 函数挂接（PrecompiledData.cpp:533-540）

从缓存 `Create()` 出每个 `asCScriptFunction` 时：

```cpp
auto* JITFunctions = JITDatabase.Functions.Find(Id);
if (JITFunctions != nullptr && !bScriptDevelopmentMode) {
    Function->jitFunction           = JITFunctions->VMEntry;
    Function->jitFunction_ParmsEntry= JITFunctions->ParmsEntry;
    Function->jitFunction_Raw       = JITFunctions->RawFunction;
}
```

函数 ID 由生成时的 `ProcessedFunctionToId` 表保证两侧一致（缓存里存的是同一批 ID）。JIT 命中的函数**跳过 bytecode 加载**（`bHasJITFunctions`）——省内存也防不一致。

### 5.3 引用重定位（PrecompiledData.cpp:2342-2394）

生成代码里 `FJitRef_Function` 全局对象的 `Pointer` 字段初始值是**生成那次运行时的旧指针**（序列化进 Cache 的 key）。模块重建完成后统一"指针修复"：

| 查找表 | 解析 | 写回 |
|---|---|---|
| `FunctionLookups` | `GetFunction({旧指针})` 按名字/签名查新函数 | `FJitRef_Function::Pointer` |
| `SystemFunctionPointerLookups` | 系统函数解析（含 caller 校验） | `FJitRef_SystemFunctionPointer::Pointer` |
| `TypeInfoLookups` | `GetTypeInfo(...)` | `FJitRef_Type::Pointer` |
| `GlobalVarLookups` | `GetGlobalVariable(...)`（全局变量内存按需重建） | `FJitRef_GlobalVar::Pointer` |
| `PropertyOffsetLookups` | `GetPropertyOffset(...)`（名字+类型 → 当前布局偏移） | `FJitRef_PropertyOffset::Offset` |

至此生成代码里所有"外部世界"的引用都指向本次运行的真实对象。之后 `FJITDatabase::Clear()` 整库丢弃（`bStaticJITTranspiledCodeLoaded` 先记下是否真的有转译代码在跑）。

## 6. 与调试/热重载的关系（为什么编辑器不用）

| 能力 | 转译路径 | 解释器路径 |
|---|---|---|
| 行断点/单步 | ✗（SUSPEND 被翻译成空；`AS_JIT_DEBUG_CALLSTACKS` 关闭时无帧信息） | ✓ |
| 数据断点（硬件） | ✓（内存层面与执行方式无关） | ✓ |
| 异常报告 | ✓（`HandleExceptionFromJIT` → 日志/调试服务器） | ✓ |
| 热重载 | ✗（`!bScriptDevelopmentMode` 才挂 jitFunction） | ✓ |
| 循环检测 | ✗（无 SUSPEND 计数）——由 `SCRIPT_ASSUME_NO_EXCEPTION`/外部看门狗兜底 | ✓ |

`WITH_EDITOR → AS_SKIP_JITTED_CODE` 的设计意图即在此：**开发期永远解释执行保调试能力，发布期整层切换到转译代码**。两份产物（Cache + 转译 cpp）同 GUID 绑定，配合 `-as-development-mode` 可以在打包版强制回到解释器排查问题。

## 6.5 脚本更新策略（游戏上线后改 as 代码）

系统没有运行时部分更新通道——打包版不读 `.as` 源码（模块从 Cache 还原），`IsValidForCurrentBuild` 也不检测源码变化。可行路径只有三条：

| 路径 | 做法 | 转译代码 | 性能 |
|---|---|---|---|
| **正式更新** | `-as-generate-precompiled-data` 全量再生成 → 新 Cache + 新 `.jit.cpp` → 重编二进制 → **成对下发** | 生效 | 转译性能 |
| **只换 Cache（紧急 hotfix）** | 再生成后只下发 `PrecompiledScript.Cache`，不动二进制 | **全部失效**（GUID 随机必不匹配，§5.1.1），整库回退解释执行缓存字节码 | 解释器性能 |
| **开发模式排查** | 打包版带 `-as-development-mode` 或删 Cache | 失效 | 解释器 + 恢复热重载（需随包分发源码） |

正式更新对"只改一小部分"的优化在 **C++ 编译层**而非转译层：`WriteOutputCode` 对内容未变的文件不重写（时间戳不变），MSVC 增量编译近似只重编改动模块所在的 `.jit.cpp`。脚本层的转译和 Cache 生成始终是全量的。

只换 Cache 的定位是**正确性兜底**：新逻辑立即生效、不需要源码、不崩溃；性能恢复必须等下一个成对更新的整包/补丁。

## 7. 值得注意的设计总结

| 设计 | 动机 |
|---|---|
| "JIT" 实为离线转译 + C++ 编译器当优化后端 | 借 MSVC/GCC 的内联、常量折叠、寄存器分配；无运行时编译的内存/线程/AV 问题 |
| FloatingStack 符号栈 + 按需物化 | 栈机字节码 → 寄存器机 C++ 的核心变换，表达式链整行内联 |
| 旧指针值当 key 的 Reference 体系 | 绕开"指针不可序列化"：生成期与运行期两次全量重定位 |
| 生成产物 GUID 配对（随机 token，非内容哈希） | Cache 与二进制严格成对；不匹配时整库回退"解释执行缓存字节码"而非崩溃。改 1 行代码重新生成 Cache 下发 → 转译代码全部失效（§5.1.1），无部分匹配 |
| 属性偏移三级策略（硬编码白名单 / 计算表达式 / 运行期解析 + 校验） | 布局稳定性与性能的折中；UE 反射类型布局漂移是真实风险 |
| 去虚化（无 override 的虚函数直连） | 消灭脚本多态最常见的间接调用，直连后可内联 |
| 每指令一个翻译类、静态自注册表 | 指令集扩展只加文件；与 `ExecuteNext` 的 switch 一一对应，可交叉验证 |
| unity 文件 + 内容不变不重写 | 转译代码体量大（每模块一 cpp，20 万行合并一个 .jit.cpp），增量产出保住迭代速度 |
| 编辑器强制 `AS_SKIP_JITTED_CODE` | 调试/热重载能力与性能路径彻底分离，互不妥协 |
| `ScriptCallNative` 与 `CallFunctionCaller` 逐行同构 | 解释器与 JIT 走完全相同的系统调用桥，行为（WorldContext 检查等）天然一致 |
| 转译后返回值走 native ABI（RAX/XMM0），跨过解释器的 valueRegister 内存往返 | 解释器的"寄存器"是 `m_regs` 结构体字段（两次内存写）；转译函数间调用/UE 直调全部真·CPU 寄存器返回，可内联（对照表见《AngelScript虚拟机架构说明》§10.1） |

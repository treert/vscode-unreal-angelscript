# AngelScript 虚拟机架构说明（字节码执行、Context/调用栈）

> 对应源码：
> - `ThirdParty/include/angelscript.h` —— `asSVMRegisters`（寄存器组）、`asEBCInstr`（指令集枚举）、`asFunctionCaller`（系统函数调用委托）、指令参数访问宏（:2030-2039）
> - `ThirdParty/source/as_context.h / .cpp` —— `asCContext`：`Execute`、`ExecuteNext` 主循环、数据栈/调用栈管理、`CallFunctionCaller`
> - `ThirdParty/source/as_callfunc.cpp` —— `CallSystemFunction`（原生 ABI 路径已被 fork 整体禁用）
> - `ThirdParty/source/as_thread.h` —— `asCThreadLocalData`（线程本地执行状态）
> - `ThirdParty/source/as_scriptfunction.cpp` —— `jitFunction` 三件套与 `JITCompile`
> - `AngelscriptCode/Private/AngelscriptManager.cpp` —— 引擎侧钩子（行回调门控、循环检测、异常桥、context 池）
>
> 路径均相对 `Engine/Plugins/Angelscript/`。

## 1. 全局视图：从字节码到执行

```
.as 源码
  │  asCParser / asCCompiler（编译期，另文分析）
  ▼
asCScriptFunction::scriptData->byteCode          asCModule::Build()
  │                                                └─ JITCompile() → jit->CompileFunction()（StaticJIT）
  │                                                     │ 编译成功
  ▼                                                     ▼
asCContext::Execute() ──── func->jitFunction != null ──► 直接执行 JIT 机器码（快路径）
  │ jitFunction == null
  ▼
while (m_status == asEXECUTION_ACTIVE)
    ExecuteNext();        ◄── 解释器主循环（慢路径，switch 分发）
```

关键结论先给出：

- **每个脚本函数有两条执行路径**：`asCScriptFunction::jitFunction` 非空走 JIT（C++ 编译的机器码），否则进解释器。入口在 `Execute()` 和每条 `asBC_CALL` 指令处二选一。
- **这个 fork 删除了 AngelScript 的"原生调用约定"支持**（`as_callfunc.cpp:397-412`，原逻辑整体 `#if 0`）。所有 C++ 绑定函数必须通过 `asFunctionCaller` 委托或 generic 接口调用——这是 UE 侧绑定层与 StaticJIT 的基石（§7）。
- **调试（行断点/单步）只存在于解释器路径**：由 `asBC_SUSPEND` 指令触发行回调（见《引擎侧断点判定细节》§2）。

## 2. 寄存器组：asSVMRegisters（angelscript.h:1422）

虚拟机没有通用寄存器堆，只有极少量"机器寄存器"，其余全部走内存栈：

| 字段 | 含义 |
|---|---|
| `programPointer` | 当前指令位置（指向当前 DWORD） |
| `stackFramePointer` | 当前函数栈帧基址（fp） |
| `stackPointer` | 数据栈顶（sp，**向下生长**） |
| `valueRegister` | 64 位暂存寄存器：算术/比较结果、原始类型返回值、跳转条件 |
| `objectRegister` / `objectType` | 对象暂存寄存器：系统函数返回的对象句柄 |
| `ctx` / `tld` | 当前 context 与线程本地数据 |

`ExecuteNext()` 开头把三个指针复制到**局部变量** `l_bc / l_sp / l_fp`（as_context.cpp:1594-1596），热循环内只操作局部副本——编译器可把它们放进 CPU 寄存器。任何可能重入 VM 的操作（CALL、系统调用、构造、行回调、异常）前后必须执行"**写回 m_regs → 操作 → 取回局部副本**"的三段式，代码中大量重复：

```cpp
m_regs.programPointer = l_bc;  m_regs.stackPointer = l_sp;  m_regs.stackFramePointer = l_fp;
CallScriptFunction(...);       // 可能嵌套执行、修改寄存器
l_bc = m_regs.programPointer;  l_sp = m_regs.stackPointer;  l_fp = m_regs.stackFramePointer;
if (m_status != asEXECUTION_ACTIVE) return;   // 嵌套调用抛了异常/结束
```

`CALLSYS` 处的注释点明了写回的另一个动机："被调函数可能用调试接口检查寄存器"——暂停时 DAP 取栈/变量读的就是 `m_regs`。

## 3. 数据栈：分块分配的本地栈

`m_stackBlocks`（`asCArray<asDWORD*>`）+ `m_stackIndex` 组成分块栈：

- 第一块大小 = `engine->initialContextStackSize`；栈不够时 `m_stackIndex++` 切到下一块，**新块大小 = 块大小 << index**（指数翻倍，`GetStackFrameSize(stackLevel) = m_stackBlockSize << stackLevel`，:5504）。跨块调用时在新块顶部留出空间 `memmove` 参数（`PrepareScriptFunction`:1508-1513）。
- `ep.maximumContextStackSize` 超限 → `SetInternalException(TXT_STACK_OVERFLOW)`（`ReserveStackSpace`:1371-1387）。
- `Execute()` 还有调用栈深度保护：顶层 `m_callStack > 20000`、函数调用 `> 10000` 都抛 "Stack overflow: potential infinite recursion detected?"（:900, :1474）——把崩溃换成可读异常。

栈帧布局（fp 相对寻址，负偏移=局部变量，正偏移=参数）：

```
高地址
┌──────────────────────────────┐
│  调用者局部变量 / 评估栈        │
├──────────────────────────────┤ ← 被调者 fp（16 字节对齐，:1508）
│  [this 指针]  （仅类方法）      │  fp[0]
│  参数区（含栈上返回值的指针槽）   │  fp[+1 ...]   ← caller 压参
│  局部变量 variableSpace        │  fp[-x]        ← PrepareScriptFunction 后 sp = fp - variableSpace
│  评估栈（表达式临时值）          │  ← sp 持续下移
└──────────────────────────────┘ 低地址
```

`PrepareScriptFunction`（:1493-1529）进函数时做三件事：对齐 sp 并搬移参数、把 `objVariablePos` 列出的**堆对象变量清零**（保证构造前为 null，异常清理才能安全 Release）、按 `variableSpace` 下移 sp。

`Prepare()`（顶层入口，:383-547）：`fp = (sp - argumentsSize - returnValueSize) & ~15`；栈上返回值时把返回值地址写进参数区第一个槽（:536-544）；`m_originalStackPointer/Index` 记录原点供同函数重入复用（:422-436）。另记录 `m_blueprintStackFrameIndex = FBlueprintContextTracker 当前脚本栈深度`（:524-531），用于把脚本调用栈对齐到 UE 的 Blueprint 调用栈（profiling/诊断用）。

## 4. 调用栈：m_callStack（与数据栈分离）

`m_callStack` 是 `asCArray<size_t>`，**每帧 11 个槽**（`CALLSTACK_FRAME_SIZE = 11`，:65），保存的是调用者的寄存器快照而非数据：

| 槽 | 内容 | 备注 |
|---|---|---|
| `[0]` | 调用者 stackFramePointer | **兼作嵌套调用标记**：`PushState()` 写 0（:1094），`RET`/`CleanStack` 据此识别"到顶了" |
| `[1]` | 调用者 `asCScriptFunction*` | `GetFunction(stackLevel)` 从这取 |
| `[2]` | 调用者 programPointer | `GetLineNumber` 从这取（**再 -1**，取调用点而非调用后，:1295-1299） |
| `[3]` | 调用者 stackPointer | |
| `[4]` | 调用者 stackIndex | |
| `[5]-[8]` | valueRegister 低/高 32 位、objectRegister、returnIsExternal | 仅 `PushState()`（嵌套执行）使用 |
| `[9]` | originalStackIndex | 仅 `PushState()` |
| `[10]` | blueprintStackFrameIndex | `GetBlueprintCallstackFrame` 从这取 |

- `PushCallState()`（:1168）先复制到局部数组 `s[6]` 再落盘，注释说明是为规避指针别名、允许编译器向量化（数据 cache 友好）。
- `PopCallState()`（:1202）恢复五组寄存器；随后若启用调试则触发 `m_stackPopCallback`——**数据断点的栈帧失效检测**就挂在这（整块失效 `oldStackIndex != m_stackIndex` / 帧内部分失效两种粒度，详见前篇 §7.3）。
- `GetCallstackSize() = 1 + len/11`：当前函数（level 0，直接读 `m_currentFunction`/`m_regs`）不占 callStack 槽。
- **PushState/PopState（嵌套执行）**：C++ 绑定函数内部再调 `Execute()` 复用同一 context 时，额外压一个 `[0]=0` 的标记帧保存全部顶层状态（包括 valueRegister/objectRegister）。`IsNested()` 通过扫 `[0]==0` 判定。

## 5. Execute()：一次顶层脚本调用（:882-1067）

```
状态检查（必须 PREPARED/SUSPENDED）→ m_status = ACTIVE
  → FScopeSetActiveContext：tld->activeContext = this
  → 调用栈深度 > 20000 ? 抛栈溢出异常
  → m_regs.programPointer == 0（首次进入）:
       ├─ 类方法：fp[0] 取 this，null → 异常
       ├─ m_executeVirtualCall && vfTableIdx != -1:
       │     objType = obj->GetObjectType()
       │     realFunc = objType->virtualFunctionTable[vfTableIdx]   ← 脚本类虚函数
       │     m_currentFunction = realFunc                            （每类型一张 VFT）
       ├─ asFUNC_IMPORTED：查 importedFunctions[].boundFunctionId
       └─ asFUNC_SCRIPT:
            ├─ jitFunction != null → FScriptExecution + 直接调 JIT，
            │    按 bExceptionThrown 决定 FINISHED / EXCEPTION，返回值写回寄存器
            └─ 否则 programPointer = byteCode，PrepareScriptFunction()
  → while (m_status == ACTIVE) ExecuteNext();
  → 收尾（fork 新增，WITH_AS_DEBUGSERVER）:
       调试开启时再补一次行回调（让监听者捕获状态变化）
       + 数据断点的整栈 stackPopCallback + TrackedReferences 泄漏校验
```

要点：

- **`m_executeVirtualCall`**：`Prepare()` 置 true（:510）。脚本类的多态不走指令，而是在**入口处查 `objType->virtualFunctionTable`** 解析出真实函数再执行。函数内的 `CALLINTF`/`CallPtr` 指令同样做 VFT 解析（接口方法还要先查 `interfaceVFTOffsets` 求接口 VFT 块的基址偏移，`CallInterfaceMethod`:1531）。
- **`Abort()` / `Suspend()` 被 fork 短路成 `return asERROR`**（:870-879）。VM 不再有协作式暂停语义——调试器的"暂停"靠行回调里忙等实现（前篇 §4），编辑器死循环防护靠循环检测（§6.3），不依赖 Suspend。
- Context **池化**：`AngelscriptRequestContext` 优先取线程本地池 `GAngelscriptContextPool`（AngelscriptManager.cpp:638-652），减少每帧创建销毁。

## 6. ExecuteNext：解释器主循环（:1592-4451）

### 6.1 指令编码

指令 = 1~3 个 `asDWORD` 的紧凑序列：

- **操作码**：首 DWORD 的最低字节（小端），`switch(*(asBYTE*)l_bc)` 分发；
- **WORD/SHORT 参数**：首 DWORD 的高 16 位（`asBC_WORDARG0/SWORDARG0`），指令最多带 3 个；
- **DWORD/INT/PTR 参数**：后续整个 DWORD（`asBC_DWORDARG/INTARG/PTRARG`，angelscript.h:2030-2039）。

switch 的 case 值**严格连续无洞**（注释：单次跳转表查找）；未用的 222-255 号被写成 `l_bc = (asDWORD*)222;`——故意制造非法写让非法字节码立刻崩溃，且阻止编译器为 switch 体积优化（:4389-4424）；MSVC 下 `default: __assume(0)`。`AS_DEBUG` 构建还会校验每条指令的实际推进长度与 `asBCTypeSize` 表一致（:4441-4449）。

### 6.2 指令类别速览

| 类别 | 代表指令 | 说明 |
|---|---|---|
| 栈/内存 | `PopPtr PshGPtr PshC4 PshV4 PshV1 PSF SwapPtr LdGRdR4` | 压栈/出栈/取址；`PSF` 取局部变量地址 |
| 控制流 | `CALL RET JMP` | |
| 条件跳转 | `JZ JNZ JS JNS JP JNP`（另有 `JLowZ` 等低位版） | 全部基于 `valueRegister` |
| 布尔测试 | `TZ TNZ TS TNS TP TNP` | 结果规范成 `VALUE_OF_BOOLEAN_TRUE` 并清零其余字节（`AS_SIZEOF_BOOL==1` 时逐字节写，volatile 防重排） |
| 算术 | `ADDi SUBi MULi DIVi ADDf ... NEGi NEGf POWi`（含 `ADDIi` 等立即数变体；溢出检测版 `as_powi`） | **三地址变量直访：源/目标都是 fp 偏移的栈帧变量，不经过求值栈**（见下） |
| 对象 | `ALLOC FREE REFCPY ObjInfo CpyVtoR4 SetV1 ...` | |
| 调用 | `CALLSYS CALLBND CallPtr CALLINTF`（见下） | |
| 特殊 | `SUSPEND ThrowException RetInfo JitEntry JNullR` | 后四个为 fork 新增/改造 |

**求值栈的角色——寄存器/变量型混合架构**：AngelScript **不是纯栈机**。算术、比较等"计算"指令不经过求值栈：

```cpp
// asBC_ADDi（:2921）——三地址变量直访，fp 负偏移直达局部变量：
*(int*)(l_fp - SWORDARG0) = *(int*)(l_fp - SWORDARG1) + *(int*)(l_fp - SWORDARG2);
// 即 x = a + b 编译成 ADDi v_x, v_a, v_b，无任何 Psh/Pop

// 比较的路径：CpyVtoR4 把变量拷进 valueRegister → JZ/JNZ/JS... 基于寄存器判定
```

求值栈（数据栈顶 `l_sp`，§3 栈帧布局的"评估栈"区）只承担三类**边界**工作：

| 场景 | 说明 |
|---|---|
| **函数调用传参** | `PshV4/PshC4/PshGPtr/PSF` 把参数压栈，被调者按 fp 正偏移读取（§3 参数区） |
| **对象/指针周转** | `SwapPtr`、ALLOC 前压对象指针、返回对象经 objectRegister |
| **系统调用** | `CallFunctionCaller` 从 `m_regs.stackPointer` 起读参数（§7.3） |

这解释了 StaticJIT 侧的对应设计（《Precompile与StaticJIT》§4.2/§4.3）：`ADDi` 翻译成 `VArg_SignedVar(0)=VArg_SignedVar(1)+VArg_SignedVar(2)`——VArg 直接映射具名 C++ 局部变量，**不经过 FloatingStack**；FloatingStack（符号表达式栈）主要服务于 `PshV4` 等**传参/压栈**指令的转译。

**调用类指令细节**：

- **`CALL`**（:1705）→ `CallScriptFunction()`：jit 命中直接执行 JIT 并弹参（:1436-1470，同样受 10000 深度限制）；否则 `PushCallState()` + 换 `m_currentFunction` + `PrepareScriptFunction()`（顺序保证栈溢出时异常处理知道现场）。
- **`RET`**（:1732）：callStack 空**或栈顶是 `[0]==0` 标记帧** → `m_status = FINISHED` 结束整个 Execute；否则 `PopCallState()` + `l_sp += w` 弹参数。
- **`CALLSYS`**（:2281）：`Function->sysFuncIntf->caller.IsBound()` ? `CallFunctionCaller()` : `CallSystemFunction()`（后者只剩 generic 路径，§7）。
- **`CALLBND`**（:2316）：跨模块导入函数——查 `boundFunctionId`，未绑定抛异常并置 `m_needToCleanupArgs`；delegate 则把 `objForDelegate` 压栈后转系统调用或接口调用。
- **`CallPtr`**（:3578）：函数指针/委托存在**局部变量**里，运行期取 `asCScriptFunction*` 再分派（脚本/系统/接口三选一）。
- **`ALLOC` / `FREE`**（:2435/2520）：脚本对象先 `AllocScriptObject` + `ScriptObject_Construct` 预初始化再调脚本构造函数；注册类型走 `CallAlloc` + 构造行为；**构造中异常会 `CallFree` 并把变量槽清零**（:2511-2512）。`FREE` 对引用计数类型为 no-op（靠 REFCPY/Release 语义）。
- **`ThrowException`**（:4342，fork 新增）：目前两个来源——switch 枚举值非法 / Unknown exception，直接 `SetInternalException` 后退出循环。
- **`RetInfo`**（:4366，fork 新增）：记录函数是从哪条 return 语句退出的，写入 `tld->ReturnInfo`（仅非 Shipping），供诊断。
- **`JitEntry`**（:3571）：**解释器中是 nop**（跳过 1+AS_PTR_SIZE 个 DWORD）。它是 JIT 机器码里"跳回解释器继续执行"的边界标记/恢复点——JIT 编译器把无法编译的片段降级回解释器时使用。

### 6.2.1 调用约定：传参与返回值（与 Lua/CIL 对照）

**参数传递——显式 push**：

```
PshV4 v_a        // --l_sp; *l_sp = fp[-v_a]     ← 一条条真压
PshV4 v_b
CALL func        // PushCallState 保存调用者 fp/pc/sp
                // 新 fp = 对齐后的 sp —— 参数"变成"被调者 fp 的正偏移
                // sp = 新 fp - variableSpace（局部变量区）
RET w            // PopCallState + l_sp += w 弹掉参数（:1752-1753）
```

**返回值——三条路径，不走参数槽**（编译器按返回类型选择）：

| 返回类型 | 被调者侧 | 调用者侧取回 |
|---|---|---|
| 原始类型（int/float/枚举） | `CpyVtoR4/R8`（:2723）把变量拷进 **valueRegister** 后 RET | `CpyRtoV4/R8`（:2738）把寄存器值拷进调用者变量 |
| 对象句柄 | 存 **objectRegister**（引用计数在 REFCPY 处理） | 直接使用 objectRegister |
| 大值对象（`DoesReturnOnStack`） | **隐藏指针参数**：调用者 Prepare 时在参数区第一个槽放返回值地址（:536-544），被调者直写那片空间 | 无需取回——对象本来就在调用者空间里 |

系统调用同构：`CallFunctionCaller` 的 `ReturnAddress` 三选一——栈上返回值槽 / `&m_regs.objectRegister` / `&m_regs.valueRegister`（:5222-5239）。

**与 Lua / CIL 的对照**：

| | CIL（C#）/ JVM | AngelScript | Lua 5 |
|---|---|---|---|
| 计算模型 | 求值栈 | **变量槽三地址**（fp 偏移，类 Lua） | 寄存器（栈帧槽位编号） |
| `a+b` 形态 | `ldloc a; ldloc b; add; stloc x`（4 条） | `ADDi v_x, v_a, v_b`（1 条） | `ADD R0 R1 R2`（1 条） |
| 传参 | 走栈（隐式栈顶） | **显式 push 指令**压栈，被调者 fp 正偏移取 | 预留连续槽，CALL 时帧窗口平移（帧重叠） |
| 返回值 | 栈顶 | **valueRegister/objectRegister/隐藏指针**（C 风格 ABI） | 写回参数槽（R[A] 窗口） |

AS 的"寄存器"与 Lua 同源——都是栈帧槽位显式寻址而非固定寄存器堆；但返回值通道学的是 **C 风格**（小值走寄存器、大值走 sret 隐藏指针，与 x64 C++ ABI 几乎一致），而非 Lua 的"返回值占参数槽"。这让 `CallFunctionCaller` 与 JIT 的返回值处理直接对齐 native 语义。

**注意 valueRegister 不是 CPU 寄存器**：它是 `asCContext` 成员结构 `m_regs` 里的一个 uint64 字段（:237）——`CpyVtoR4 → RET → CpyRtoV4` 是两次跨指令的**内存往返**；指令边界处值必然落回 `m_regs`（switch 分发会破坏调用者保存寄存器，无法 pin 在 RAX）。解释器里真正常驻 CPU 寄存器的只有 `l_bc/l_sp/l_fp` 三个指针。真·寄存器返回只存在于转译路径（见 §10.1）。

### 6.3 SUSPEND：行回调与循环检测的宿主（:2397-2432）

```cpp
case asBC_SUSPEND:
    if (CanEverRunLineCallback)                                   // 全局静态门控（编辑器且调试/数据断点开）
        if (ShouldAlwaysRunLineCallback                           // 单步中：无条件
            || (m_currentFunction->module && ...->hasBreakPoints))// 模块级门控
            m_lineCallback(this);                                 // 写回寄存器后回调（可能暂停忙等）
    if (++m_loopDetectionCounter > 100000)                        // 每 ~10 万行
    {
        m_loopDetectionCallback(this);                            // 引擎侧超时检测
        if (m_status != asEXECUTION_ACTIVE) return;
        m_loopDetectionCounter = 0;
    }
```

- 编译器在**每条语句边界**插入 `SUSPEND`——这就是"行"的物理形态，也是断点行号吸附（`FindNextLineWithCode`）能成立的底层原因。
- 三层门控让非调试态的 SUSPEND 只剩一条 `case`+`break` 的开销（前篇 §2.1）。
- 循环检测：`AngelscriptLoopDetectionCallback`（AngelscriptManager.cpp:4069）用 `m_loopDetectionTimer` 对比 `EditorMaximumScriptExecutionTime`，超时杀掉脚本；`m_loopDetectionExclusionCounter` 可临时豁免（如调试监视求值）。断点暂停时前篇 §4 会把 timer 置 -1 防误报。

## 7. 系统调用桥：caller 委托制（fork 最大改动之一）

### 7.1 原生 ABI 路径被整体移除

上游 AngelScript 的 `CallSystemFunction` 按平台 ABI（x64：RCX/RDX/R8/R9 + XMM）把 VM 栈上的参数搬进寄存器直接 call C 函数（`as_callfunc_x64_msvc.asm` 等）。本 fork：

```cpp
// as_callfunc.cpp:397-412 —— 生效代码只有 10 行
int CallSystemFunction(int id, asCContext *context)
{
    ...
    if (sysFunc->caller.IsBound())
        return context->CallFunctionCaller(descr);       // ① caller 委托
    if (callConv == ICC_GENERIC_FUNC || ICC_GENERIC_METHOD)
        return context->CallGeneric(descr);              // ② generic 兜底
    context->SetInternalException("Native calling convention support is disabled. "
                                  "Make sure you're passing a correct Caller.");
    return 0;                                            // ③ 其余一律异常
    // 以下 ~330 行原生路径（对象指针弹栈、baseOffset、try/catch C++ 异常、
    // 返回值寄存器回填、参数清理）全部 #if 0
}
```

`as_callfunc_x64_msvc.cpp` 同样被 `#if 0` 关闭；`PrepareSystemFunction` 的原生返回值布局计算也停在 `#elif 0`。相应地，`RegisterGlobalFunction / RegisterObjectMethod / RegisterObjectBehaviour` 全系列接口都增加了 `asFunctionCaller caller = nullptr` 参数（angelscript.h:714+）。

### 7.2 asFunctionCaller（angelscript.h:672）

```cpp
using FunctionCallerPtr = void(*)(asFUNCTION_t Method, void** Parameters, void* ReturnValue);
using MethodCallerPtr   = void(*)(asMETHOD_t  Function, void** Parameters, void* ReturnValue);
```

绑定方（`AngelscriptBinds`/`Bind_*.cpp` 的模板代码生成）为每个 C++ 函数生成一个 caller：caller 收到**函数指针 + 参数指针数组 + 返回值地址**三样东西，自己负责按 C++ 语义调真实函数。ABI 细节完全由绑定层的模板代码（编译期实例化）承担，而不是运行期汇编桩。

### 7.3 CallFunctionCaller（as_context.cpp:5159-5279）

```
① WorldContext 安全检查（编辑器）：
     asTRAIT_USES_WORLDCONTEXT 的函数被 BlueprintThreadSafe 脚本调用 → 直接异常
     当前无 WorldContext 且配置开启报错 → 异常
② thiscall：从栈上取对象指针（StackArgs[0]），null → 异常
③ 可选元数据首参：asEFirstParamMetaData::ScriptFunction/TypeInfo 额外塞一个参数
④ 返回地址选择：
     栈上返回    → 参数区里的返回值槽地址
     对象句柄    → &m_regs.objectRegister
     其余        → &m_regs.valueRegister
⑤ 参数打包（按 descr->parameterOffsets 预计算偏移）：
     对象/引用   → 传指针值（*reinterpret_cast<void**>）
     原始值类型  → 传栈上槽位的地址（caller 自行解引用拷贝）
     ttQuestion  → 指针值 + typeId 槽地址（两个参数位）
⑥ tld->activeFunction = descr → 调 caller（type 1 函数 / type 2 方法）→ 恢复
⑦ return descr->totalSpaceBeforeFunction;   ← 调用方按此弹栈
```

动机（推测+代码佐证）：① 跨平台一致（不再需要每平台汇编桩）；② 绑定签名可校验；③ **StaticJIT 可以直接内联/直调 C++ 函数**——caller 是普通 C++ 函数指针，JIT 代码里就是一次 native call；④ UE 侧能在调用前后插 WorldContext/线程安全检查。

## 8. 异常处理与栈展开

### 8.1 异常的产生与记录

`SetInternalException`（:4460）：置 `m_status = EXCEPTION`，记录函数 id、行号/列号（`GetLineNumber` 返回 `line | (column << 20)`，20 bit 行号 + 12 bit 列号），触发应用层异常回调。解释器中每条可能抛异常的指令后都有 `if (m_status != ACTIVE) { 写回寄存器; return; }`，逐层从 `ExecuteNext` 退到 `Execute`。

### 8.2 栈展开：CleanStack（:4519-4544）

```
CleanStack()
  ├─ CleanStackFrame()            ← 当前帧：DetermineLiveObjects 按
  │     "变量 declaredAtProgramPos <= 当前 pc" 判定已声明的存活对象，逐个析构/释放
  ├─ m_status = EXCEPTION         ← 之后的帧按异常语义清理
  └─ while (callStack 非空 且 栈顶不是 [0]==0 嵌套标记):
        PopCallState() → CleanStackFrame()   ← 逐层退栈清理，直到嵌套执行边界
```

`m_needToCleanupArgs`：在"参数已压栈、调用本身失败"的场景（未绑定函数、接口空指针等）提示清理器把参数区的临时对象也释放掉。

### 8.3 UE 异常 ↔ 脚本异常的双向桥

引擎统一入口（AngelscriptManager.cpp:3490 附近）：

```cpp
if (tld->activeExecution != nullptr)      // 正在跑 JIT 代码
    tld->activeExecution->bExceptionThrown = true;   // 标记后 JIT 快速退栈
else if (tld->activeContext != nullptr)   // 正在跑解释器
    tld->activeContext->SetException(...);           // 标准脚本异常
```

- **JIT 路径**没有解释器的逐指令检查点，靠 `FScriptExecution::bExceptionThrown` 标志：JIT helper（`FStaticJITFunction::SetNullException` 等，StaticJITHeader.cpp:103+）置位后层层快速 return，最终在 `Execute`/`CallScriptFunction` 的调用点被检查（:967, :1451），置 `m_status = EXCEPTION`。
- C++ `check`/`ensure` 类异常在 caller 委托内发生时也走这条桥（原生路径的 `try/catch → HandleAppException` 已随 `#if 0` 移除）。

## 9. 线程本地执行状态：asCThreadLocalData（as_thread.h）

每线程一份（`thread_local` 指针）：

| 字段 | 用途 |
|---|---|
| `activeContext` | **解释器路径**的当前 context（`FScopeSetActiveContext` 在 `Execute`/`Unprepare` 时设置） |
| `activeExecution` | **任意路径**的当前 `FScriptExecution`（RAII 链，含 `prevExecution/prevContext`） |
| `primaryContext` | 线程主 context |
| `activeFunction` | 当前正在执行的**系统函数**（`CallFunctionCaller` 设置，供 `GetStackTrace`/诊断） |
| `ReturnInfo` | 最近一次 return 语句编号（`asBC_RetInfo` 写入） |

`FScriptExecution`（as_context.h:246-283）是 fork 的核心抽象之一：**任何一次脚本执行**（解释器 Execute、JIT 直调、UE 反射直调 JIT）都用这个 RAII 对象在 tld 里压一个执行帧，构造时 `activeContext = nullptr`（JIT 期间没有"活跃 context"概念，异常桥由此区分路径）。`EntryJitFunction` 记录入口 JIT 函数供栈回溯。

## 10. JIT 集成（VM 视角）

> StaticJIT 编译器本身的实现（`Private/StaticJIT/`）是另一篇文档的主题，这里只看 VM 侧挂点。

- **注册**：`Engine->SetJITCompiler(StaticJIT)`（AngelscriptManager.cpp:381）。
- **编译时机**：`asCModule::Build()` 尾部 `JITCompile()`（as_module.cpp:303）→ 逐函数 `jit->CompileFunction(this, &jitFunction)`（as_scriptfunction.cpp:1501-1514）。**Build 完成即全部编译**——不是惰性 JIT。
- **三个函数指针**（as_scriptfunction.h:339-341）：`jitFunction`（带 VM 栈帧入口，签名 `(FScriptExecution&, asDWORD* l_fp, asQWORD* outValue)`）、`jitFunction_ParmsEntry`（`(Execution, void* Object, void* Parms)`——UE Parms 结构入口）、`jitFunction_Raw`（无参裸入口）。
- **VM 内入口**：`Execute()` 顶层与 `asBC_CALL` → `CallScriptFunction()` 都先查 `jitFunction`（§5、§6.2）；命中则完全绕开解释器与 asCContext 的栈管理，返回值按类型写回 object/value 寄存器，`l_sp += totalSpaceBeforeFunction` 弹参。
- **UE 直调入口**：`UASFunction_JIT` 系列（ASClass.cpp:636+ 的 `MakeRawJITCall_*`）——**UObject 反射调用脚本函数时直接进 JIT 机器码**，连 asCContext 都不创建（线程安全函数另有 generic 路径）。虚拟调用先 `ResolveScriptVirtual` 查 VFT 再取 `jitFunction`。
- **JIT → 解释器降级**：`asBC_JitEntry` 占位（§6.2）。
- **调试盲区**：JIT 路径没有 `SUSPEND`/行回调——断点/单步只在解释器路径逐行生效；数据断点（硬件断点）不受影响，因为它们工作在内存写入层面。

### 10.1 返回值通道：解释器 vs 转译路径的"寄存器"差异

| 路径 | 返回值通道 | 是否真 CPU 寄存器 |
|---|---|---|
| 解释器 → 解释器 | `m_regs.valueRegister`（asCContext 结构体字段，§6.2.1） | ✗——`CpyVtoR4→RET→CpyRtoV4` 是两次跨指令内存往返，指令边界必然落回 `m_regs` |
| 解释器收到系统函数返回 | caller thunk 内部真实经 RAX，随即写入 `&m_regs.valueRegister`（内存） | 半程（RAX→立刻落内存） |
| 转译函数 ↔ 转译函数 | 普通 C++ 调用约定（native ABI），可被内联 | ✓ RAX/XMM0 |
| UE 反射 → 转译函数 | `MakeRawJITCall_ReturnValue<T>` 模板直接 return | ✓ 全程 native |

`Execute()` 顶层那个 `asJITFunction(Execution, l_fp, &outValue)` 的指针出参只是**VM 汇合点的边缘协议**；转译函数彼此之间的调用不经过它，走的就是 C++ ABI。同一份字节码的 `RET` 语义，在两条路径隔着"模拟寄存器（内存）→ 物理寄存器"的跃迁——这是转译后性能收益的重要来源之一（另一来源是表达式内联，见《Precompile与StaticJIT》§4.3）。

## 11. UE 侧定制汇总（对照上游 AngelScript 2.3x）

| 改动 | 位置 | 动机 |
|---|---|---|
| 删除原生调用约定，改 `asFunctionCaller` 委托 | as_callfunc.cpp、angelscript.h:672 | 跨平台一致；绑定层模板生成调用代码；StaticJIT 可直接 native call |
| `FScriptExecution` + tld 执行帧链 | as_context.h:246、as_thread.h | UE↔脚本异常双向桥；任意位置（含 UE 直调 JIT）可重入脚本执行 |
| Build 后全量 StaticJIT + 三形态入口 | as_module.cpp:303、ASClass.cpp | 脚本性能接近 C++；UE 反射调用绕开 VM |
| `CanEverRunLineCallback`/`ShouldAlwaysRunLineCallback` 静态门控 + `module->hasBreakPoints` | as_context.cpp:67-68、ExecuteNext SUSPEND | 行回调非调试态零开销（前篇 §2.1） |
| `Abort()`/`Suspend()` 短路为 asERROR | as_context.cpp:870-879 | 暂停语义由调试器忙等承担；避免外部线程撕裂状态 |
| 循环检测（10 万行计数 + 超时计时） | SUSPEND 内 + AngelscriptManager.cpp:4069 | 编辑器死循环防护，且只在非 Shipping 生效 |
| 调用栈深度上限 20000/10000 + 明确异常 | Execute / CallScriptFunction | 无限递归给脚本er 可读报错而非崩溃 |
| `m_stackPopCallback` | PopCallState | 数据断点栈帧失效检测（前篇 §7.3） |
| `m_blueprintStackFrameIndex`（每调用帧槽 [10]） | Prepare / callStack | 脚本栈帧 ↔ `FBlueprintContextTracker` 调用栈对齐 |
| `asBC_RetInfo` → `tld->ReturnInfo` | 指令级 | 诊断"函数从哪个 return 退出" |
| `asBC_ThrowException` | 指令级 | switch 非法枚举等显式异常 |
| `MovedToNewThread()`、context 线程池 | as_context.cpp:214、Manager:638 | UE 任务系统线程迁移 + 池化复用 |
| `AS_REFERENCE_DEBUGGING`（TrackedReferences） | as_context.h:239 | 开发期校验引用计数清理正确性（Execute 收尾 ensure） |

## 12. 与调试链路的接口（承前篇）

调试器暂停时读到的所有状态都来自 VM 的这些只读接口，且因为"暂停=游戏线程忙等"（前篇 §4），无需快照：

- `GetAddressOfVar(var, level)`：`stackFramePointer - 变量 stackOffset`（变量表存负偏移，as_scriptfunction 的 `variables`）；
- `IsVarInScope`：`declaredAtProgramPos` 与当前 `programPointer` 比较（:4571）；
- `GetLineNumber(level)`：level 0 用 `m_regs.programPointer`，上层用调用帧 `[2]` 再减 1（调用点行号）；
- `GetCallstackSize / GetFunction / GetThisPointer / GetStackFrame`：§4 的调用帧布局；
- 一切成立的前提是解释器**在回调前已把 l_bc/l_sp/l_fp 写回 m_regs**（§2）——这是 VM 与调试器之间隐式的协约。

## 13. 值得注意的设计总结

| 设计 | 动机 |
|---|---|
| 寄存器极简（3 指针 + 2 值寄存器）+ 全栈式寻址 | 解释器实现简单、JIT 编译器映射容易（局部变量即栈偏移）；代价是访存密集，靠 JIT 弥补性能 |
| 主循环局部副本 + 显式写回 | 热路径寄存器化；写回点同时服务"嵌套重入"与"调试读取"两个需求 |
| 调用栈与数据栈分离（m_callStack 是数组，不放在脚本数据栈上） | 调用栈长度可预测、免于数据栈溢出影响回溯；栈帧快照式保存让 RET/异常展开都是 O(1) 弹帧 |
| 分块指数增长的数据栈 + 跨块参数搬移 | 单块大栈浪费内存；小块起步 + 翻倍兼顾深递归与占用 |
| SUSPEND 即"行"、即调试探针、即循环检测采样点 | 一个指令同时挂三件事，非调试态三层门控后近乎零成本 |
| caller 委托替代平台 ABI 桩 | 可移植、可校验、JIT 可直调；代价是绑定层复杂度和每次调用的参数指针数组 |
| 异常双层机制（解释器 status 检查 / JIT bExceptionThrown 标志） | 解释器有天然检查点；JIT 用标志位快速退栈，两种路径在 Execute/CallScriptFunction 汇合 |
| 虚函数在入口/VFT 查表而非指令层 | 脚本类多态退化为"取真实函数指针"，JIT 与解释器可共用同一解析逻辑 |

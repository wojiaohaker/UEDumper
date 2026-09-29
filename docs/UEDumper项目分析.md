# UEDumper 项目分析文档

> 分析对象：`e:\ReverseWorkSpace\UEDumper`
> 项目作者：Spuckwaffel（开源，MIT 许可证）
> 版本：Dumper `RELEASE_1_11_FINAL`（v1.11）

---

## 一、项目概述

**UEDumper** 是一个"一体化"的 **Unreal Engine（虚幻引擎）SDK 转储工具（Dumper）**，支持从 **UE 4.19 一直到 UE 5.4/5.7** 的各个引擎版本。它的核心用途是：在游戏运行时读取目标进程的内存，解析虚幻引擎反射系统（UObject / UStruct / UEnum / UFunction / FProperty 等）的内部数据结构，最终生成可用于二次开发的 **C++ SDK**、**MDK**，并支持 **Dumpspace** 格式导出。

除了静态转储，项目还内置一个基于 **ImGui** 的富交互 **GUI**，包含：

- **SDK 生成器与编辑器**：可视化浏览所有 Package、Struct、Class、Enum、Function。
- **Live Editor（实时编辑器）**：在游戏运行时以生成的 SDK 结构为参照，浏览并读写游戏内存（如遍历整个 `UWorld`）。
- **工程持久化**：可将转储结果保存为 `.uedproj` 文件，离线（OFFLINE MODE）加载复用。

> ⚠️ 项目明确指出**不能开箱即用**：用户需要自己对目标游戏做逆向，找到 `GObjects`、`GNames`、`GWorld` 等偏移/特征码，并按需实现 FName 解密函数与内存读写驱动。该工具仅供学习研究用途。

### 关键特性一览

| 特性 | 说明 |
|------|------|
| 多引擎版本支持 | UE 4.19 ~ UE 5.4+，通过宏切换内部结构布局，无需改动核心代码 |
| 富 GUI | 基于 Dear ImGui + DirectX 11 + Win32 |
| SDK / MDK 生成 | 生成可直接用于 C++ 工程的头文件与实现文件 |
| Dumpspace 支持 | 导出社区通用格式 |
| Live Editor | 运行时内存浏览与修改，带刷新节流 |
| 高度缓存 | 大量使用 `unordered_map` 缓存 FName/UObject，保证转储性能 |
| 可插拔驱动 | 内存读写逻辑集中在 `driver.h`，方便替换为绕过反作弊的自定义驱动 |

---

## 二、技术栈与构建

- **语言**：C++20（`<LanguageStandard>stdcpp20</LanguageStandard>`），大量使用 `std::unordered_map`、concepts 风格模板、`std::ranges`、结构化绑定等。
- **平台/工具集**：Windows x64，MSVC `v143`，Unicode 字符集。
- **工程文件**：`UEDumper.sln` + `UEDumper.vcxproj`。
- **图形/界面**：Dear ImGui（内置于 `Frontend/ImGui`），DX11 + Win32 后端，字体以头文件内嵌（`Frontend/Fonts`）。
- **第三方库**：
  - `nlohmann/json`（`Resources/Json/json.hpp`）——工程序列化、Dumpspace 输出。
  - 自实现 `AES`（`Resources/AES`）——加密资源处理。
  - `DirectXTex/WICTextureLoader`——纹理加载。
- **CI**：GitHub Actions（`.github/workflows/build-verification.yml`），在 `windows-latest` 上用 `msvc-dev-cmd` 配置 x64 后执行 `msbuild`，仅做**编译验证**。

---

## 三、整体架构

项目采用清晰的**分层架构**，可分为四层：

```
┌─────────────────────────────────────────────────────┐
│  Frontend 表现层 (ImGui 窗口 / 渲染主循环)              │
│  HelloWindow · DumpProgress · PackageWindow ·         │
│  PackageViewerWindow · LiveEditor · LogWindow ...     │
├─────────────────────────────────────────────────────┤
│  Engine 引擎层                                         │
│  ├─ Core      核心逻辑与缓存 (EngineCore/ObjectsManager)│
│  ├─ UEClasses Unreal 反射类的手工定义 (UObject/UStruct…)│
│  ├─ Generation SDK / MDK 代码生成                       │
│  ├─ Live      实时内存编辑                              │
│  └─ Userdefined 用户配置 (宏/偏移/类型/结构体覆盖)        │
├─────────────────────────────────────────────────────┤
│  Memory 内存抽象层 (Memory 包装类 + driver.h 驱动实现)   │
├─────────────────────────────────────────────────────┤
│  Settings / Resources (EngineSettings · JSON · AES)   │
└─────────────────────────────────────────────────────┘
```

入口在 [`UEDumper.cpp`](file:///e:/ReverseWorkSpace/UEDumper/UEDumper/UEDumper.cpp) 的 `main()`：初始化 ImGui 辅助类、内存、纹理与各窗口，加载编译期宏设置，然后进入渲染主循环，根据当前阶段（Hello → Dumping → Package/LiveEditor）渲染不同界面。

### 1. Engine/Userdefined —— 用户配置层（"为你的游戏定制"）

这是使用者**唯一需要修改**的目录，通过编译期宏驱动引擎行为：

- [`UEdefinitions.h`](file:///e:/ReverseWorkSpace/UEDumper/UEDumper/Engine/Userdefined/UEdefinitions.h)：设置 `UE_VERSION`（默认 `UE_4_27`）以及影响结构体布局的一堆开关，如 `WITH_CASE_PRESERVING_NAME`、`UE_BLUEPRINT_EVENTGRAPH_FASTCALLS`、`FUOBJECTITEM_SIZE`、`GOBJECTS_XOR_ECRYPTION_KEY`、`UE_FNAME_OUTLINE_NUMBER` 等。每个开关都对应虚幻引擎源码里的编译宏，直接决定 FName/UObjectItem/UFunction 等结构的字段偏移与大小。
- [`Offsets.h`](file:///e:/ReverseWorkSpace/UEDumper/UEDumper/Engine/Userdefined/Offsets.h)：定义 `Offset` 结构与 `OffsetFlags`（地址 / 特征码跟随 / 特征码直接 / Dumpspace / LiveEditor 标记）。用户在此登记 `OFFSET_GNAMES`、`OFFSET_GOBJECTS`、`OFFSET_GWORLD` 等，默认占位 `0xDEADBEEF`（配合宏 `SHOW_README_IF_OFFSETS_ARE_VALUE` 提醒用户必须先配置）。
- [`Datatypes.h`](file:///e:/ReverseWorkSpace/UEDumper/UEDumper/Engine/Userdefined/Datatypes.h)：定义基础类型在 SDK 中的显示名（`int32_t`/`uint8_t`…）与 `getSize()` 尺寸计算。
- `StructDefinitions.h` / `FeatureFlags.h`：手工覆盖或补充引擎中缺失/被改动的结构体成员。

> 设计要点：所有版本差异都收敛到这些宏里，`structs.h`、`Core.cpp` 等核心代码通过 `#if UE_VERSION ...` 条件编译适配，从而实现"不改核心即可支持新版本"。

### 2. Engine/UEClasses —— 反射类手工建模

[`UnrealClasses.h`](file:///e:/ReverseWorkSpace/UEDumper/UEDumper/Engine/UEClasses/UnrealClasses.h)（约 1245 行）用 C++ 手工重现了虚幻引擎的反射对象模型：`UObject`、`UStruct`、`UClass`、`UFunction`、`UEnum`、`UProperty`、`FField`/`FProperty`（4.25+ 的属性系统重构）等，字段布局随 `UE_VERSION` 条件编译变化。

一个巧妙细节：`UObject` 的首字段是 `objectptr`，被用作"伪 vtable"位置——缓存对象时把真实游戏指针写入 `object + 0`，从而让每个缓存的 `UBigObject` 都能反查原始指针（见 ObjectsManager 注释与 `getFullName()` 等辅助方法）。

### 3. Engine/Core —— 核心逻辑与缓存

- [`Core.h`](file:///e:/ReverseWorkSpace/UEDumper/UEDumper/Engine/Core/Core.h) 中 `EngineCore` 是**纯静态类**，负责：
  - FName 缓存 `FNameCache`、全名字符串到 UObject 指针的 `fullStringCache`；
  - Package 容器 `packages`、`packageObjectInfos`；
  - 结构体/枚举/函数的生成入口：`generateStructOrClass`、`generateEnum`、`generateFunctions`；
  - 成员"烹饪"（cook）：`cookMemberArray` 处理继承、padding、缺失字节块的对齐；
  - 用户覆盖：`overrideStruct` / `overrideStructMembers` / `createStruct` / `createEnum`（生成前）与 `runtimeOverrideStructMembers`（运行时）；
  - 工程持久化：`saveToDisk`（写 `.uedproj`）、`loadProject`（离线加载）、`generateStructDefinitionsFile`；
  - `FNameToString`：跨版本的 FName → 字符串解析（含分块 namePool 定位、可选解密、UTF 转换），是全项目最复杂、注释最密集的函数之一。

- [`ObjectsManager.h`](file:///e:/ReverseWorkSpace/UEDumper/UEDumper/Engine/Core/ObjectsManager.h)：管理所有 UObject 的批量缓存。
  - 按版本区分 `FFixedUObjectArray`（<4.20）与 `FChunkedFixedUObjectArray`（≥4.20）两种 `GObjects` 布局。
  - 转储流程：`copyGObjectPtrs`（扫描对象指针表）→ `copyUBigObjects`（把每个 UObject 拷贝进预分配大缓冲区 `UBigObject`，单对象上限 `UOBJECT_MAX_SIZE = 0x150`）。
  - 两种缓存态 `CS_SDKGEN`（严格用大缓冲）与 `CS_RUNTIME`（Live Editor 时按需增量缓存新对象）。
  - 4.25+ 额外的 `FFieldManager`（缓存 FField / FFieldClass，`FFIELD_CT = 400000`）。
  - 支持 XOR 加密指针解密 `decryptPointer`（`GOBJECTS_XOR_ECRYPTION_KEY`）。

- `EngineStructs.h`：定义转储产物的中间数据结构——`Package`、`Struct`（含 `definedMembers`/`undefinedMembers`/`cookedMembers`）、`Member`、`Function`、`Enum`、`fieldType`、`ObjectInfo`。每个结构都带 `toJson()/fromJson()`，用于工程序列化。

### 4. Engine/Generation —— SDK / MDK 代码生成

- [`SDK.h`](file:///e:/ReverseWorkSpace/UEDumper/UEDumper/Engine/Generation/SDK.h) `SDKGeneration`：按 Package 输出 C++ SDK 头/源文件，支持 feature flags、包排序（`packageSorter.h`）、基础类型生成，可选写入 `static_assert` 尺寸校验（`WRITE_STATIC_ASSERT_TESTS`）。
- `MDK.h` `MDKGeneration`：生成 MDK（一种面向函数偏移/参数补全的进阶 SDK 形式）。
- `BasicType.h`：内置类型与容器（TArray/TMap/FString 等）模板定义。

### 5. Engine/Live —— 实时编辑器后端

`LiveMemory.cpp/.h` 支撑 Live Editor 在运行时按 SDK 结构解析内存对象，配合 `Frontend/Windows/LiveEditor.cpp` 绘制成员树、节流刷新（默认 500ms）并允许改写非指针成员。

### 6. Memory —— 内存抽象层

- [`Memory.h`](file:///e:/ReverseWorkSpace/UEDumper/UEDumper/Memory/Memory.h)：全项目**唯一**的内存访问入口，静态类，提供 `load()`（按进程名/PID 取基址与 PID）、模板化 `read<T>()`/`write<T>()`、`patternScan()` 特征码扫描，并统计读写次数。
- [`driver.h`](file:///e:/ReverseWorkSpace/UEDumper/UEDumper/Memory/driver.h)：**驱动实现层**，默认用 `ReadProcessMemory`/`WriteProcessMemory` + `CreateToolhelp32Snapshot` 取基址。用户若要绕过反作弊，只需替换此处 `_read`/`_write`/`_getBaseAddress` 的实现而不动调用方。`_read` 失败时会自动降级重试（每次少读 10 字节）。

### 7. Frontend —— 表现层

`Frontend/Windows` 下每个窗口是一个静态类，负责一块 UI：

- `HelloWindow`：启动界面，输入工程名与目标进程名，选择加载已有 `.uedproj`（进入离线模式，禁用 Live Editor）。
- `DumpProgress`：转储进度，用 `std::async` 在后台线程串起整条转储流水线（见下）。
- `PackageWindow` / `PackageViewerWindow`：Package 列表与结构体浏览，支持点击成员/继承链跳转、搜索、独立导航历史、成员编辑。
- `LiveEditor`：运行时内存浏览与修改。
- `LogWindow`：分级日志（0~4，级别越低信息越多）。
- `TopRowButtons`：顶部操作按钮（保存工程、生成 SDK、生成 StructDefinitions 等）。
- `IGHelper` / `StrucGraph` / `Texture` / `Fonts` / `ImGui`：ImGui 封装、图形绘制、纹理与字体资源。

### 8. Settings / Resources

- `EngineSettings`：运行时设置与宏快照（`loadMacros()` 把编译期宏读入可查询变量）、工程名/工作目录/离线标志管理、与 JSON 互转。
- `Resources`：`AES`（加密）、`Json`（nlohmann）、`Dumpspace`（社区格式导出）。

---

## 四、核心工作流程

### 转储流水线（DumpProgress 后台线程）

参见 [`DumpProgress.cpp`](file:///e:/ReverseWorkSpace/UEDumper/UEDumper/Frontend/Windows/DumpProgress.cpp)：

```
EngineCore() 初始化 (解析 GNames/GObjects 偏移)
      ↓  失败 → 报错终止
ObjectsManager() 初始化对象数组
      ↓
copyGObjectPtrs   扫描 UObject 指针表 → 大缓冲
      ↓
copyUBigObjects   把每个 UObject 内存拷贝为 UBigObject
      ↓
cacheFNames       缓存全部 FName → 字符串
      ↓
generatePackages  遍历 UObject 生成 Package/Struct/Enum/Function
      ↓
setSDKGenerationDone() + 开启 Live Editor
```

每一步都检查 `CopyStatus` 与 `ObjectsManager::CRITICAL_STOP_CALLED()`，任何关键内存错误都会记录错误信息并中止，UI 弹出警告图标。转储结果全程缓存到内存中的 `EngineStructs::Package` 向量，供 GUI 浏览与后续 SDK/MDK/工程保存使用。

### FName 解析要点

`EngineCore::FNameToString` 针对 UE 4.19~4.22（无 chunk 的 `TStaticIndirectArrayThreadSafeRead`）与 4.23+（`FNameEntry` 分块 + `FNameEntryId`）分别实现，处理 `WITH_CASE_PRESERVING_NAME`、`UE_FNAME_OUTLINE_NUMBER`、`GNAMES_POOL_OFFSET`、可选解密（`FName_decryption.h`）以及 UTF-16/Ansi → UTF-8 转换，是全项目版本兼容复杂度的集中体现。

---

## 五、设计亮点

1. **宏驱动的多版本兼容**：把引擎版本差异完全收敛到 `UEdefinitions.h` + 条件编译，核心逻辑"一次编写、多版本适配"，扩展新版本成本低。
2. **缓存优先**：FName、UObject、FField 全部预缓存到大缓冲区并用哈希表索引，避免反复跨进程读内存，转储又快又稳定（README 强调"大量 caching"）。
3. **关注点分离**：`Memory`（接口）与 `driver.h`（实现）解耦，方便替换底层读写以对抗反作弊；`Engine` 与 `Frontend` 解耦，逻辑不依赖 UI。
4. **可插拔的用户配置**：偏移、解密、结构体覆盖都放在 `Userdefined`，不改引擎内核。
5. **序列化能力**：核心数据结构统一 `toJson/fromJson`，支持 `.uedproj` 工程保存与离线复用。
6. **文档化良好**：核心文件大量注释直接引用虚幻引擎 GitHub 源码链接，便于对照逆向。

## 六、潜在风险与改进点

- **代码规模大、单函数复杂**：`Core.cpp` 约 1800 行、`UnrealClasses.h` 约 1245 行，FName/成员烹饪逻辑分支繁多，可读性与维护成本较高。
- **内存安全依赖约定**：`UBigObject` 固定 `0x150`、结构体尺寸靠用户宏正确性保证；用户覆盖成员大小错误会导致 SDK 错位与 Live Editor 崩溃（README 明确警告）。
- **全局静态状态密集**：`EngineCore`/`ObjectsManager` 大量 `inline static` 全局，线程安全依赖"后台单线程转储 + UI 只读"的隐含约定。
- **平台/用途限制**：仅 Windows x64、依赖直接进程内存读写；面对带反作弊的商用游戏需自行实现驱动。
- **合法合规**：属内存逆向工具，仅供学习研究，使用需注意法律与游戏服务条款边界。

---

## 七、目录结构速查

| 目录 | 职责 |
|------|------|
| `Engine/Userdefined` | 用户配置：UE 版本宏、偏移、数据类型、结构体覆盖 |
| `Engine/UEClasses` | 虚幻反射类（UObject/UStruct/UFunction/FField…）手工建模 |
| `Engine/Core` | `EngineCore` 核心逻辑/缓存、`ObjectsManager` 对象缓存、FName 解密 |
| `Engine/Generation` | SDK / MDK 代码生成、包排序、基础类型 |
| `Engine/Live` | Live Editor 运行时内存后端 |
| `Memory` | 内存访问抽象（Memory）与驱动实现（driver.h） |
| `Frontend/Windows` | 各 ImGui 功能窗口 |
| `Frontend/ImGui·Fonts·Texture` | 界面渲染、字体、纹理资源 |
| `Settings` | 运行时引擎设置与工程信息 |
| `Resources` | AES、nlohmann::json、Dumpspace 导出 |

---

## 八、上手使用流程（面向使用者）

1. **确定 UE 版本**，在 `UEdefinitions.h` 设置 `UE_VERSION` 及相关结构开关。
2. **逆向找偏移**：`GNames` / `GObjects` / `GWorld` 等，填入 `Offsets.h`（名字须唯一，`OFFSET_GNAMES`/`OFFSET_GOBJECTS` 为必需）。
3. （如加密）在 `FName_decryption.h` 实现解密函数，置 `USE_FNAME_ENCRYPTION = TRUE`。
4. （如有反作弊）在 `driver.h` 替换读写/取基址实现。
5. 编译运行 → Hello 窗口输入工程名与精确进程名 → 开始转储 → 浏览 Package / 生成 SDK·MDK / 保存 `.uedproj` → 可选进入 Live Editor 实时读写内存。

---

*本文档基于对仓库源码的静态分析生成，供架构理解与二次开发参考。*




结论先说：**不支持开箱即用**。无论引擎版本还是 AngelScript 改造，都需要你自己做适配和逆向才能工作。下面分两部分说明依据。

## 1. 引擎版本：5.8.3 超出了 UEDumper 的支持范围

我读取了两边的关键信息：

- 目标引擎 `d:\UnrealEngine\UE_Angelscript_SC\Engine\Build\Build.version`：
  ```json
  { "MajorVersion": 5, "MinorVersion": 8, "PatchVersion": 3, "CompatibleChangelist": 55116800, "BranchName": "UE5" }
  ```
  确认是 **UE 5.8.3**。

- UEDumper 的版本枚举 [`UEdefinitions.h`](file:///E:/ReverseWorkSpace/UEDumper/UEDumper/Engine/Userdefined/UEdefinitions.h#L50-L67) 最高只定义到：
  ```cpp
  #define UE_5_07  16 // Fixes FFieldClass::Id offset
  ```
  也就是说**根本没有 `UE_5_08` 这个宏**，README 里官方声称的范围也只是 4.19～5.3/5.4（第 9、22 行）。

关键点在于 UEDumper 的工作方式：它不自动探测版本，而是**依赖为每个版本硬编码的内部反射结构偏移**（`UObject` / `UClass` / `UStruct` / `UProperty` / `FField` / `FUObjectItem` 等），这些布局在 5.x 各小版本之间一直在变——连 `UE_5_07` 都专门标注了"修复 FFieldClass::Id 偏移"。5.8 相对 5.4 之后很可能又改了这些结构，所以：

- 你不能直接把 `UE_VERSION` 设成一个不存在的 5.8；
- 需要自己逆向出 5.8.3 下 `GObjects / GNames / GWorld` 的偏移或签名（[`Offsets.h`](file:///E:/ReverseWorkSpace/UEDumper/UEDumper/Engine/Userdefined/Offsets.h#L106-L115) 目前全是占位 `0xDEADBEEF`），并核对 `UnrealClasses.cpp/.h` 里反射结构体的成员偏移是否还匹配。

## 2. AngelScript 改造本身：一般不是主要障碍，但要验证

- AngelScript 那套（UnrealEngine-Angelscript 类 fork）主要是**新增一层脚本反射/生成机制**，把 AS 脚本暴露成 UCLASS/UPROPERTY。这些在 dump 里通常表现为普通的 package / UClass，UEDumper 能当作正常对象遍历——**这块通常不需要特殊改动**。
- 真正需要确认的是这个被"其他工作室改过"的 fork 是否动过 **UObject 核心反射结构的内存布局**（比如给 UClass/UProperty 加了自定义成员）。README 第 56、74-75 行明确警告：像 Fortnite/PUBG 这类魔改引擎的 fork，如果改了内部结构，必须自己在 `UnrealClasses.cpp` 里修正，否则 dump 会错位甚至崩溃。

## 建议的落地步骤

1. 先反编译目标游戏的 `World Partition` 无关，重点找 `GNames`、`GObjects`（以及 live editor 需要的 `GWorld`）的地址/签名。
2. 用 UE 5.8.3 源码（就在 `d:\UnrealEngine\UE_Angelscript_SC\Engine\Source\Runtime\CoreUObject`）对照 `UObject/UClass/UStruct/Property/Field` 的真实成员偏移，核对并按需修改 UEDumper 里 [`UnrealClasses.h/.cpp`](file:///E:/ReverseWorkSpace/UEDumper/UEDumper/Engine/UEClasses) 与相关宏。
3. 在 `UEdefinitions.h` 里新增 `UE_5_08` 宏，并把代码中所有 `#if UE_VERSION >= UE_5_0x` 的分支检查一遍，确认新版本走的是正确路径。
4. 若该 fork 确实改动了反射结构，额外在 `StructDefinitions.h` 里覆盖对应成员。

一句话总结：**5.8.3 版本 + 被魔改的 fork，两者叠加决定了 UEDumper 无法开箱支持**；AngelScript 脚本层一般无碍，但引擎 5.8 的内部结构偏移和该工作室可能的自定义改动，都需要你手动逆向适配后才能正常 dump。

需要的话，我可以直接在 UE 5.8.3 源码里帮你核对 `UObject`/`UClass`/`FField` 等结构体的成员偏移，并与 UEDumper 现有定义做一份差异清单。
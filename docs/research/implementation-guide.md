# 过加固对抗 — 完整实现文档

> 分支基线：`docs/anti-hardening-research` / 上游 JingMatrix/LSPosed（Vector）master `efb82883`
> 依据：`anti-hardening-bypass.md`（检测面）、`video-bypass-details.md`（视频绕过细节）、`execution-plan.md`（排期）
> 日期：2026-10-02

---

## 1. 目标与范围

### 1.1 目标
在本仓库（Vector 框架）上实现一套**可开关、可验证**的反检测改造，使得：
1. 框架自身在目标进程内**不留可枚举特征**（类名、dex 字符串、so 名、线程名）；
2. **不在 libart.so / 任何系统库代码段上做 inline hook** 的基线保持不变并可审计；
3. 为模块提供**统一的消迹能力**（maps 过滤、线程名、dex 混淆），而不是每个模块各写一套；
4. 每一项改造都有**对应的检测项与验证命令**，可回归。

### 1.2 边界（不做什么）
| 层级 | 归属 | 本框架是否负责 |
|---|---|---|
| root 挂载痕迹 / mountinfo 连续性 | ZygiskNext + Shamiko / 内核模块（Nohello 类） | ❌ 不负责 |
| 时序侧信道（truncate supercall / statx） | 内核模块层 | ❌ 不负责 |
| Play Integrity / keybox | TrickyStore + PIF | ❌ 不负责 |
| 应用列表隐藏 | HMA | ❌ 不负责 |
| **注入与 hook 痕迹、dex/类特征、maps 内容** | **Vector 框架** | ✅ 负责 |
| **模块级消迹 API** | **Vector 框架** | ✅ 负责 |

---

## 2. 现状代码基线

### 2.1 注入链路（`zygisk/src/main/cpp/module.cpp`）

```
postAppSpecialize
 ├─ ipc_bridge.RequestAppBinder(nice_name)      // daemon 决定作用域，不在作用域直接 unload
 ├─ ipc_bridge.FetchFrameworkDex() → (dex_fd, size)   // 框架 dex 经 binder IPC 传 fd，不落盘
 ├─ ipc_bridge.FetchObfuscationMap() → ConfigBridge::obfuscation_map()
 ├─ LoadDex()   → InMemoryDexClassLoader(ByteBuffer, systemClassLoader)
 ├─ InitArtHooker() → lsplant::Init(env, init_info_)
 ├─ InitHooks() → MakeDexFileTrusted(mCookie) + RegisterResourcesHook/HookBridge/NativeApiBridge
 ├─ SetupEntryClass() → obfs_map["org.matrix.vector.core."] + "Main"
 └─ forkCommon(...)  → Java 侧接管
```

### 2.2 已有的反检测能力（可直接复用）

| 能力 | 位置 |
|---|---|
| dex 类签名随机化（8 组特征 → 等长随机串） | `daemon/src/main/jni/obfuscation.cpp` |
| 随机签名经 IPC 下发到每个进程 | `zygisk/.../module.cpp:347` → `ConfigBridge` |
| 全链路使用随机签名取类/入口 | `hook_bridge.cpp:958`、`resources_hook.cpp:53`、`jni_bridge.h:45`、`ipc_bridge.cpp:518` |
| 不解析 `/proc/self/maps` 取库基址，改问 linker | master HEAD #959（`elf_image` / `symbol_cache`） |
| Binder `execTransact` 用 JNI 层替换，非 inline hook | `ipc_bridge.cpp:511-545` |
| dex 不落盘（InMemoryDexClassLoader + memfd） | `module.cpp:LoadDex` |

### 2.3 缺口清单（本次要实现的部分）

| # | 缺口 | 证据 |
|---|---|---|
| ~~G1~~ | ~~`obfuscateDex` 无 Java 调用方~~ | **判断有误（已更正）**：`ObfuscationManager.kt` 存在且已接线 —— `FileSystem.readDex()` 调 `obfuscateDex()`、`loadModule()`/`getPreloadDex()` 均按 `ConfigCache.state.isDexObfuscateEnabled` 走混淆，该开关 release 版默认 `true`（`DaemonState.kt:15`，`!BuildConfig.DEBUG`）。此前结论源于一次被 `head` 截断的 grep，勿据此改代码 |
| ~~G2~~ | LSPlant 生成代理类名硬编码 `Vector_` / `Dobby` | ✅ **已修复（T4，commit `70462655`）** —— 改为每进程随机，`strings libzygisk.so` 已无这两个字面量 |
| ~~G3~~ | 第三方模块 dex 未走混淆 | **判断有误（部分更正）**：模块 dex 本身**会**走混淆（`FileSystem.loadModule()` 对 `classes*.dex` 逐个 `readDex(obfuscate=true)`）。真实缺口是**签名表只覆盖 8 组框架前缀**，模块自带的遗留特征串漏网 → ✅ **已修复（T1，commit `9b1357ed`）** |
| G4 | `init_info_.inline_hooker` 仍挂 Dobby → 存在 inline hook 痕迹风险 | `module.cpp:init_info_`（`HookInline/UnhookInline`），**待办（T5）** |
| G5 | 框架自身线程名、so 名、ClassLoader 可被枚举 | 无随机化逻辑 |
| G6 | 无 maps/smaps/map_files 内容过滤能力 | 仅"自己不读 maps"，但**目标 App 会读** |
| G7 | 无 solist / link_map 摘除 | — |

---

## 3. 总体设计

```
                      ┌─────────────── daemon / manager 侧（启动时一次性） ───────────────┐
                      │ T1  生成随机签名表 (8 组框架签名 + N 组模块签名)                  │
                      │ T2  对 framework dex / 每个 module dex 调 obfuscateDex           │
                      │ T3  签名表随 FetchObfuscationMap 下发                             │
                      └───────────────────────────┬───────────────────────────────────────┘
                                                  │ binder IPC
                      ┌───────────────────────────▼───── 目标进程（zygisk 注入后） ──────┐
                      │ T4  LSPlant 生成类名随机化（去 Vector_/Dobby）                    │
                      │ T5  inline_hooker 去 Dobby 化（零 .text 改写）                    │
                      │ T6  框架线程 comm / so 名 / ClassLoader 名随机化                  │
                      │ T7  maps|smaps|map_files 输出过滤（libc 层，按开关启用）          │
                      │ T8  solist + link_map 摘除（延迟到注入完成后，带死锁保护）        │
                      └──────────────────────────────────────────────────────────────────┘
```

**配置开关**（`ConfigBridge` 扩展 + manager 配置项，默认全开，可逐项关闭以便二分定位问题）：

```
vector.hide.dex_obfuscation      = true    // T1/T2/T3
vector.hide.generated_class_name = true    // T4
vector.hide.no_inline_hook       = true    // T5
vector.hide.thread_and_so_name   = true    // T6
vector.hide.maps_filter          = true    // T7
vector.hide.solist_prune         = false   // T8（默认关：有死锁风险，进阶项）
```

---

## 4. 实现任务

### T1 — 生成随机签名表（daemon 侧） ✅ 已落地（commit `9b1357ed`，分支 `code/T1-extend-signatures`）

**已做**：在 `daemon/src/main/jni/obfuscation.cpp` 的 `signatures` 表追加 3 组模块侧特征串：
- `Lorg/lsposed/`（按旧版 API 写的模块）
- `Lcom/swift/sandhook/`（模块自带 SandHook）
- `Lme/weishu/epic/`（模块自带 Epic）

`regen()` 保证替换串与被替换串**等长**（dex 结构不变），`FileSystem.loadModule()` 会用同一张表改写 `moduleClassNames`，因此这三组同时作用于 dex 字符串与模块类名。

**剩余（T1b，未做）**：自动扫描模块 dex 字符串表动态扩充签名。难点是时序——签名表必须在任何 dex 被混淆之前生成，动态扩充需要「先扫全部模块 dex 再统一混淆」的两阶段加载，改动面较大，列为后续课题。

**验收**：`adb logcat -s Vector | grep -i obfuscat` 可见新增三组 `xxx => yyy`；重启后不同。

### T2 — 接线 `ObfuscationManager` ✅ 上游已实现，无需改动

实际代码（此前判断有误，以代码为准）：
- `daemon/.../utils/ObfuscationManager.kt` 已存在（`external fun obfuscateDex/getSignatures`）；
- `FileSystem.readDex(inputStream, obfuscate)` → `ObfuscationManager.obfuscateDex(memory)`；
- 调用方：`FileSystem.loadModule(apkPath, obfuscate)`（模块全部 `classes*.dex`）、`FileSystem.getPreloadDex(obfuscate)`（`framework/vector.dex`）；
- 开关：`ConfigCache.state.isDexObfuscateEnabled`，默认 `!BuildConfig.DEBUG`（release 版开启）；
- `FileSystem.loadModule` 还会用 `getSignatures()` 改写 `moduleClassNames`。

**注意**（源码注释已警示）：JNI 侧 `mmap` 必须 `MAP_SHARED`，`MAP_PRIVATE` 会因 COW 读到零页导致 slicer 失败。

### T3 — 签名表下发 ✅ 已有（`module.cpp` FetchObfuscationMap → ConfigBridge）

待补：`.at()` 取签名前加 `find()` 兜底，避免 map 缺 key 抛异常导致注入失败（当前 `SetupEntryClass`/`resources_hook`/`hook_bridge`/`ipc_bridge` 均为直接 `.at()`）。

### T4 — LSPlant 生成类名随机化 ✅ 已落地（commit `70462655`，分支 `code/T4-randomize-generated-names`）

实现（`zygisk/src/main/cpp/module.cpp`）：
- 新增 `RandomIdentifier(len)`：首字母必为字母，其余字母/数字，产出合法 Java 标识符；
- `init_info_` 由 `const` 改为可写，去掉 `generated_class_name = "Vector_"` / `generated_source_name = "Dobby"` 两个字面量；
- 新增 `RandomizeGeneratedNames()`，在**两条注入路径**（`postAppSpecialize`、`postServerSpecialize`）里、拿到混淆表之后、`InitArtHooker()` 之前调用；
- 名字存于 `VectorModule` 成员（`InitInfo` 只持有 `string_view`，必须保证生命周期覆盖 `lsplant::Init()`）；
- **不查 map 的固定 key**（那会把 key 重新变成字面量），改为每进程本地随机。

**验证**：`strings libzygisk.so | grep -c '^Dobby$'` → 0；`^Vector_$` → 0。
**残留**：LSPlant 上游自带的默认名 `LSPHooker_` 仍在二进制中（上游字面量，运行时不使用）。

### T5 — 去除 inline hook 痕迹（G4）
**原则**：视频结论 + 本仓库路线 = **不在任何系统库 .text 上做 inline patch**。
**改动**：
1. 审计 `init_info_.inline_hooker`（Dobby）的实际调用路径——LSPlant 主要在初始化/取符号阶段可能使用；统计 `HookInline` 命中次数并在 debug 日志输出；
2. 对 libart 侧一律走 `art_symbol_resolver` + **entry_point 替换**（LSPlant 默认路径），禁用对 `libart.so` 的 inline；
3. 需要 PLT 级拦截时用 **LSPlt**（改 GOT，不动 .text），而非 Dobby；
4. 开关 `no_inline_hook=true` 时，若仍有代码请求 inline hook → 返回失败并落日志（便于发现残留调用方）。

**验收**：`adb shell cat /proc/<pid>/maps` 中 `libart.so` 可执行段**只有一段**（COW 分段消失）；Hunter 的 libart CRC 项不报。

### T6 — 线程名 / so 名 / ClassLoader 随机化
**改动**：注入完成（`forkCommon` 返回）后，在 native 侧统一处理：
```cpp
// 1) 线程 comm：框架创建的所有线程（logcat/bridge/ipc）
pthread_setname_np(pthread_self(), RandomComm());      // 禁止 gmain/gum-js-loop/lspd 等特征
// 2) so 名：框架库加载时走 memfd + dlopen 匿名路径（Zygisk 侧已匿名，确认 maps 无 .so 后缀路径）
// 3) ClassLoader：InMemoryDexClassLoader 已匿名；确认 toString() 不含可识别前缀
```
**验收**：`ls /proc/<pid>/task/*/comm` 无框架特征名；`ls -l /proc/<pid>/fd` 无框架 dex/so 路径（readlink）。

### T7 — maps / smaps / map_files 输出过滤（G6）
**背景**：视频三明确指出 VMA 有**三个**用户态展示口，只处理 maps 会被 smaps / `/proc/self/map_files` 抓到。
**改动**：新增 `native/src/hide/maps_filter.cpp`，对 libc 的 `openat`/`read`/`fopen` 走 **LSPlt PLT hook**（不动 .text）：
- 匹配路径 `/proc/self/maps`、`/proc/self/smaps`、`/proc/self/map_files`、`/proc/<pid>/maps`（pid=自身）；
- 过滤规则：丢弃包含框架特征串（随机签名、`liblspd`、`/data/adb/modules/*`）的行；
- **只过滤目标 App 的读取**（按 uid/包名白名单开关），system_server 与自身进程不过滤，避免破坏系统功能；
- 提供 `vector.hide.maps_filter.extra_packages` 供用户追加。

**风险**：PLT hook libc 对高频调用有性能开销；必须先做 perf 基线（`openat` 调用频次）。

**验收**：目标 App 内 `cat /proc/self/maps` 无框架行；同时 `smaps` 与 `map_files` 也干净（三条命令逐一验证）。

### T8 — solist + link_map 摘除（进阶，默认关）
**背景**：视频三：只摘 solist 不摘 `r_debug->link_map` 一样被检测；摘 link_map 在 dlopen 持锁路径上可能**死锁**。
**实现要点**：
- 时机：必须在 `postAppSpecialize` **全部注入动作完成后**（LSPlant 已初始化、dex 已加载、hook 已安装）再摘，避免 linker 正在遍历；
- 摘除方式：遍历 `solist` 找到自身 soinfo，从链表摘除；再从 `r_debug->r_map` 摘除对应 link_map；
- **保护**：加 `dlopen` 计数守卫——若摘除期间检测到并发 dlopen（linker 锁被持有），跳过摘除并记日志；
- 默认关闭（`solist_prune=false`），仅在确认稳定性后按需开启。

**验收**：目标 App 内遍历 linker solist/link_map 无框架条目；连续启动 50 次无 ANR/死锁。

---

## 5. 验证方案（每项对应一条命令）

| 任务 | 验证命令 / 工具 | 通过标准 |
|---|---|---|
| T2/T3 | `adb shell run-as <pkg> cat /proc/self/maps \| grep -iE 'lspd\|xposed\|vector'` | 无输出 |
| T2 | 目标进程 dex strings 扫描（frida/objection 或 Hunter 的 dex 项） | 无 Xposed/LSPosed 字样 |
| T4 | Hunter「dex 扫描」+ 自写 dex strings 脚本 | 无 `Vector_`/`Dobby` |
| T5 | `adb shell cat /proc/<pid>/maps \| grep libart` | 可执行段仅 1 段，无 rwxp |
| T5 | Hunter libart CRC 项 | 不报「libart 被修改」 |
| T6 | `adb shell for t in /proc/<pid>/task/*; do cat $t/comm; done` | 无框架特征线程名 |
| T6 | `adb shell ls -l /proc/<pid>/fd \| xargs -n1 readlink` | 无框架 dex/so 路径 |
| T7 | 目标 App 内分别读 maps / smaps / map_files | 三处均无框架行 |
| T8 | 自定义 linker 遍历 so（或 Hunter 的 so 列表项） | 无框架 so 条目 |
| 回归 | Native Detector / Hunter / Ruru / Momo / 牛头人 | 逐项记录，与改造前对比 |

**A/B 方法**（沿用既有约定）：同一台设备、同一套检测器，先刷未改造构建取基线，再刷改造后构建对比，**每次只开一个开关**以定位副作用。

---

## 6. 排期与风险

| 阶段 | 任务 | 依赖 | 风险 / 回退 |
|---|---|---|---|
| S1 | T1+T2+T3（dex 混淆接线） | 构建环境就绪（子模块/local.properties） | slicer 改写失败 → 保留原 dex（现有代码有 `needs_obfuscation` 短路，直接返回原 fd） |
| S2 | T4+T6（命名随机化） | S1 | 类名不一致导致注入失败 → 开关一键关闭回退 |
| S3 | T5（去 inline hook） | 审计 Dobby 调用点 | 某些 Android 版本必须 inline → 保留 per-version 白名单 |
| S4 | T7（maps 过滤） | LSPlt 可用 | 性能/稳定性 → 默认按包白名单，非目标 App 不启用 |
| S5 | T8（solist 摘除） | S1-S4 稳定 | 死锁 → 默认关闭 + dlopen 守卫跳过 |

**回退策略**：所有改造都在 `vector.hide.*` 开关后；任一开关关闭即回到当前基线行为，不改动既有代码路径。

---

## 7. 检测面 → 实现任务映射（总表）

| 检测面（来自视频三） | 对应任务 | 归属 |
|---|---|---|
| dex 字符串 / ClassLoader 枚举 | T1/T2/T3 | 框架 ✅ |
| LSPlant 生成类名 `Vector_`、来源 `Dobby` | T4 | 框架 ✅ |
| libart CRC / COW 分段 / rwxp | T5 | 框架 ✅ |
| 线程 comm 特征（gmain/gum-js-loop 等） | T6 | 框架 ✅ |
| fd readlink 暴露 dex/so 路径 | T6 | 框架 ✅ |
| maps / smaps / map_files 三通道 | T7 | 框架 ✅ |
| solist / r_debug link_map | T8 | 框架（进阶）✅ |
| 匿名可执行段 | T5（减少 trampoline）+ Zygisk 层 | 框架 + Zygisk |
| 挂载 / mountinfo 连续性 | — | ZygiskNext/Shamiko/内核模块 ❌ |
| 时序侧信道（truncate/statx） | — | 内核模块 ❌ |
| Play Integrity / keybox | — | TrickyStore/PIF ❌ |
| 应用列表 | — | HMA ❌ |

---

## 8. 落地纪律

1. 文档与代码分支隔离：本文档在 `docs/anti-hardening-research`；代码改动必须在 `explore/anti-hardening-bypass` 上**再开子分支**；
2. 每个任务一个 commit，commit message 标注对应任务号（T1…T8）与检测项；
3. 每个任务合入前必须跑通 §5 对应验证命令，并在 `execution-plan.md` 附录 A 打勾；
4. 临时调试输出不进 git；真机刷写前备份 boot 镜像。

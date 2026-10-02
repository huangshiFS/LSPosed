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
| G1 | `obfuscateDex` JNI 已实现但**无 Java 调用方**（无 `ObfuscationManager` 类） | 全仓库 grep 无 `ObfuscationManager` 引用；`daemon/src/main/kotlin` 下只有 `Cli.kt`/`VectorDaemon.kt` |
| G2 | LSPlant 生成代理类的名字硬编码 `Vector_`、来源名 `Dobby` → dex 里可扫 | `module.cpp:init_info_`（`generated_class_name = "Vector_"`, `generated_source_name = "Dobby"`） |
| G3 | 第三方**模块 dex**（非框架 dex）未走混淆 | obfuscation 只覆盖 8 组框架签名 |
| G4 | `init_info_.inline_hooker` 仍挂 Dobby → 存在 inline hook 痕迹风险 | `module.cpp:init_info_`（`HookInline/UnhookInline`） |
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

### T1 — 生成随机签名表（daemon 侧）
**改动**：`daemon/src/main/jni/obfuscation.cpp`（已有 `regen()`），新增导出 `getSignaturesForModules()`
- 现有 8 组：`Lde/robv/android/xposed/`、`Landroid/app/AndroidApp`、`Landroid/content/res/XRes`、`Landroid/content/res/XModule`、`Lio/github/libxposed/api/Xposed`、`Lorg/matrix/vector/core/`、`Lorg/matrix/vector/nativebridge/`、`Lorg/matrix/vector/service/`；
- **追加**：扫描已启用模块 APK 的 dex 字符串表，自动提取形如 `Lxxx/ModuleMain;`、`Lde/robv/...`、`Lorg/lsposed/...`、`XposedBridge`、`XC_MethodHook` 的用户自定义特征串一并纳入 `signatures`（长度对齐规则复用 `regen`，保证 dex 结构不变）；
- 约束：**替换串与被替换串长度必须相等**（现有代码已校验 `out.length() != original.length()` 并 LOGE）。

**验收**：`adb logcat | grep ObfuscationManager` 可见 N 条 `xxx => yyy`；重启后签名表不同。

### T2 — 接线 `ObfuscationManager`（补 G1）
**改动**：新增 `daemon/src/main/kotlin/org/matrix/vector/daemon/utils/ObfuscationManager.kt`
```kotlin
object ObfuscationManager {
    external fun getSignatures(): Map<String, String>          // T1 产出
    external fun obfuscateDex(memory: SharedMemory): SharedMemory?  // 已实现的 JNI
}
```
调用点（两处）：
1. **框架 dex 下发前**：`FetchFrameworkDex` 的服务端实现处，把 dex 写入 `SharedMemory` → `obfuscateDex()` → 再把结果 fd 返回给 zygisk；
2. **模块 dex 下发前**：模块 APK 解出 dex → 同样过一遍。

**注意**（源码注释已警示）：JNI 侧 `mmap` 必须 `MAP_SHARED`，SharedMemory 用 `MAP_PRIVATE` 会被 COW 层吞掉内容导致 slicer 失败。

**验收**：目标进程内 `adb shell cat /proc/<pid>/maps` 无模块路径；dex 内 grep 不到 `XposedBridge`。

### T3 — 签名表下发（已有，补模块签名）
`module.cpp:347` 的 `FetchObfuscationMap` 已打通；只需保证 T1 扩展后的 map 也走同一通道，并在 `SetupEntryClass`/`resources_hook`/`hook_bridge` 用 `.at()` 前做 `find()` 兜底（避免缺 key 抛异常导致注入失败）。

### T4 — LSPlant 生成类名随机化（去 `Vector_` / `Dobby`）
**改动**：`zygisk/src/main/cpp/module.cpp` 的 `init_info_` 改为运行时生成：
```cpp
const lsplant::InitInfo init_info_{
    .inline_hooker = ...,                     // 见 T5
    .art_symbol_resolver = ...,
    .generated_class_name = RandomName("Vector_"),   // 形如 "aX7kQ_"，每次 boot 随机
    .generated_source_name = RandomName("Dobby"),    // 关键: dex 里不再出现 "Dobby"
};
```
`RandomName()` 从 `ConfigBridge::obfuscation_map()` 取，保证 daemon 与进程内一致（Java 侧若需引用代理类也要走 map）。

**验收**：`adb shell` 进进程 `/proc/<pid>/maps` 抓 dex → `strings` 无 `Vector_`/`Dobby`；Hunter 的 dex 扫描项不报。

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

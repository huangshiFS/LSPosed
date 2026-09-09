# LSPosed 对抗企业版/海外加固 — 探索研究笔记

> 分支：`explore/anti-hardening-bypass`（基于 JingMatrix/LSPosed master `efb82883`，canary-3111）
> 日期：2026-09-09

---

## 1. 视频要点提炼

三个视频均为同一 UP 主「一只勇敢的小佳」（mid 1886803835），圈内知名的 LSPosed 改版作者：

| 视频 | 日期 | 核心信息 |
|---|---|---|
| BV1XHnEzkEPL「小佳修改的最新lsposed」 | 2025-09-27 | 改版 LSPosed 直接过海外大部分加固：**hunter、DexProtector、scb 类、appdemo** 等；关键手法是「**无 libart 的 hook**」；与原版 API 几乎一致、兼容性高；支持 HMA（Hide-My-Applist）系统级 hook |
| BV1ipdpBwEeh「最强内核模块，最强小工具，最强lsposed」 | 2026-04-18 | 演示性质：内核模块（KernelSU 系）+ 小工具 + 改版 LSPosed 组合，强调「极致手法」——即检测对抗已下沉到内核层 |
| BV1WEovBjEVh「检测总览」 | 2026-04-26 | 39 分钟系统梳理**检测方的全部思路**——这是理解"怎么过"的前提：先穷尽检测面，再逐面消迹 |

核心结论：**过加固 = 穷尽检测面 × 逐面无痕迹化**。小佳改版的公开技术路线与 JingMatrix/LSPosed（Vector 重构版）的开源实现高度重合，本仓库即为最好的研究样本。

---

## 2. 检测面全景（检测方能查什么）

结合「检测总览」视频主题与公开检测器实现（Native Detector / Hunter / Ruru / 各加固厂 RASP），检测面分五层：

### 2.1 Java 层
| 检测点 | 原理 |
|---|---|
| 特征类探测 | `Class.forName("de.robv.android.xposed.XposedBridge")`、`Lorg/lsposed/lspd/...`、libxposed 包名 |
| 堆栈回溯 | 抛异常遍历 stacktrace，找 `XposedBridge`、重复 `ZygoteInit` 帧、hook 代理类帧 |
| ClassLoader 异常 | 模块 dex 注入产生的额外 PathClassLoader / InMemoryDexClassLoader |
| 方法属性异常 | hooked 方法 access_flags 变化（`kAccCompileDontBother`、native 化）、`Method.invoke` 行为异常 |

### 2.2 Native 层（/proc 与内存）
| 检测点 | 原理 |
|---|---|
| maps 扫描 | `/proc/self/maps` 中 `liblspd.so`、`lsplant`、`/data/adb/modules/*/zygisk/*.so`、frida/gadget 字样 |
| smaps COW 取证 | r-xp 段被 inline hook 改写 → 产生 private dirty 页，smaps 中 `Private_Dirty` 非零 |
| 匿名可执行页 | Dobby/Frida trampoline 落在匿名 rwxp/r-xp 页，正常进程极少有 |
| prologue 校验 | 关键函数头 16~32 字节哈希比对，inline hook 必破坏 |
| .text 完整性 | 内存 .text 与磁盘 so 逐字节比对 |
| 指令模式 | ARM64 上 Dobby 的 `LDR Xn,[PC,#off]; BR Xn` 特征序列 |

### 2.3 系统层
| 检测点 | 原理 |
|---|---|
| 全系统进程扫描 | 遍历 `/proc/[pid]/cmdline`，找 `lspd`、`zygiskd`、`rezygiskd` 守护进程 |
| 文件标记 | `/data/adb/modules/.zygisk`、`/data/adb/modules/zygiskNext`、`lspd` 目录 |
| dlopen 探测 | `dlopen("libzygisk.so", RTLD_NOLOAD)` 判断库是否已在内存 |
| 应用列表 | 装了 LSPosed Manager、HMA、Magisk 等 → HMA 存在的意义 |
| Play Integrity | 设备完整性 verdict（海外 App 必查） |

### 2.4 加固壳特有检测（企业版/海外 RASP）
- **Promon / Build38 / DexProtector / Appdome / 爱加密 / 梆梆企业版**：
  - guard 网络互相校验（hook 一个 guard 触发其他 guard 报警）
  - ART 内部结构校验：ArtMethod entry_point 是否指向非预期位置、JNI 函数表是否被换
  - 壳自身代码 CRC：对壳 so 做段校验，inline hook 壳函数立刻被发现
  - 反调试 + TracerPid + 端口扫描（frida 27042/27043）
  - 行为层：hook 后时延异常、三相退出（killProcess + exit + halt 并发，单 hook 无法阻止）

---

## 3. 原版 LSPosed 的可检测特征 → 本仓库（JingMatrix/Vector）的对应消迹

本仓库 master 已是 JingMatrix 的重构版（内部代号 Vector），阅读源码可确认以下反检测机制**已经落地**：

| # | 检测面 | 原版特征 | 本仓库的消迹实现 | 代码位置 |
|---|---|---|---|---|
| 1 | 特征类探测 | `Lde/robv/android/xposed/`、`Lio/github/libxposed/api/Xposed`、`Lorg/matrix/vector/` 等硬编码包名出现在 dex 字符串表 | **DEX 类签名随机化**：daemon 在内存中用 slicer 原地改写模块 dex，把 8 组特征签名替换为**每次 boot 随机生成**的等长随机类名（保持长度不变避免破坏 dex 结构） | `daemon/src/main/jni/obfuscation.cpp` |
| 2 | maps 扫描 | 解析 `/proc/self/maps` 找库加载基址，暴露自身且被扫到特征路径 | **改为直接问 linker**：通过 linker 内部结构（solist）拿 soinfo，不再解析 maps 文件 | master HEAD commit `efb82883` "Ask the linker where a library is loaded, not the maps file" (#959) |
| 3 | libart inline hook | 老版本在 libart.so 导出函数上打 inline hook → smaps COW / prologue 校验必死 | **无 libart hook**：全部 Java hook 走 LSPlant 的 ArtMethod entry_point 替换 + 动态生成代理 dex（JNI 桥），native 侧只用 LSPlt 对 linker/libnativeloader 做 PLT hook（不改动 .text） | `native/src/jni/hook_bridge.cpp`、`external/lsplant`、`external/lsplt` |
| 4 | 注入面 | 老版注入所有进程，特征扩散 | zygisk 入口按 UID/包名精确过滤（isolated、app_zygote、relro 进程直接跳过），未选中的应用进程内**完全没有框架代码** | `zygisk/src/main/cpp/module.cpp` |
| 5 | hook 调度痕迹 | 全局回调表无线程安全，异常路径留栈帧 | `HookItem` 原子化 backup + priority multimap，hook 调用走 JNI 直接转发，Java 堆栈中不留 Xposed 帧 | `native/src/jni/hook_bridge.cpp:31-91` |

「无 libart 的 hook 直接过 hunter」——视频里的这句话在源码层面的含义就是第 2、3 条：**不在 libart.so 代码段上做任何 inline patch**，hunter 的 smaps 取证和 prologue 校验全部落空。

---

## 4. 完整对抗路线（过企业版/海外加固的分层方案）

单一工具无法过企业版加固，业界共识是分层叠加：

```
Layer 0  Root 隐藏        KernelSU/Magisk + Shamiko（mount 命名空间级隐藏，对目标 App 不可见）
Layer 1  Zygisk 痕迹      ZygiskNext/NeoZygisk + 排除列表；守护进程名随机化
Layer 2  框架消迹         本仓库 Vector：dex 签名随机化 + 无 libart hook + solist 查询（§3）
Layer 3  应用列表         HMA（Hide-My-Applist）对目标 App 隐藏 Manager/Magisk 等
Layer 4  设备完整性       PlayIntegrityFix/Fork + keybox（海外 App 的 Play Integrity verdict）
Layer 5  目标 App 专项    Xposed 模块逐点中和：File.exists 过滤、maps 读取过滤、
                          检测函数返回值改写、上报包拦截（参考 modules.lsposed.org 的
                          io.github.xalsace.qqbypass 七层结构）
Layer 6  内核兜底         小佳视频2的「内核模块」：SUSFS/内核级隐藏处理 Layer 0-1 剩下的
                          侧信道（挂载点、proc 节点、syscall 痕迹）
```

**过企业版加固（乐固/爱加密/梆梆企业版）的额外要点**：
1. 壳有代码 CRC → 绝不 inline hook 壳的 so；hook 点在壳解密落地后的 Java/ART 层（LSPlant entry_point 替换天然满足）
2. guard 网络互相校验 → 要么全部同时中和，要么只 hook「检测结果的消费点」（if 分支/上报函数）而不是检测函数本身
3. 三相退出 → hook `Process.killProcess` / `System.exit` / `Runtime.halt` 三个入口的**调用判定处**，而非拦截退出原语
4. 时机：模块在 `handleLoadPackage` 早期就要完成检测点中和，早于壳的 RASP 初始化轮询

**过海外加固（DexProtector/Promon/Appdome/LIAPP）的额外要点**：
1. Play Integrity 是硬门槛，keybox 被封是日常 → 需可替换指纹的 PIF 方案
2. Promon 偏行为检测 → 减少 hook 数量、避免高频轮询型模块
3. DexProtector native bridge 有完整性校验 → 同壳 CRC 原则，只动 Java 层

---

## 5. 本分支后续实验计划

- [ ] 初始化子模块并完整构建：`git submodule update --init && ./gradlew :zygisk:assembleRelease`
- [ ] 在 Pixel 6 / OnePlus 9 Pro 上刷入本分支构建，跑 Native Detector / Hunter / Ruru 基线
- [ ] 对比原版 LSPosed 1.9.2 与本版的检测命中项差异（验证 §3 表格）
- [ ] 用红果/抖音（douyin 工作区）实测：SSL pinning bypass 模块在本框架下是否触发壳检测
- [ ] 研究 obfuscation 覆盖面是否含第三方模块 dex（`Lde/robv` 之外的模块自定义特征类）
- [ ] 评估把 hook_bridge 的 trampoline 从匿名页挪入已存在 RX 段的可行性（针对匿名可执行页扫描）

---

## 6. 附录 A：检测项 → 消迹手段对照核查表

真机验证时按此表逐项核对（检测工具：Native Detector / Hunter / Ruru / Momo / Duck Detector）：

| 检测项 | 检测工具典型实现 | 期望通过的消迹手段 | 验证状态 |
|---|---|---|---|
| `Class.forName("de.robv...XposedBridge")` | Java 反射探测 | obfuscation.cpp dex 签名随机化 | ☐ 待真机验证 |
| 异常堆栈含 Xposed 帧 | `new Throwable().getStackTrace()` | JNI 直接转发，不留 Java 帧 | ☐ |
| maps 含 `liblspd`/模块路径 | 读 `/proc/self/maps` 字符串匹配 | solist 查基址 + 注入面 UID 过滤 | ☐ |
| smaps r-xp 段 `Private_Dirty > 0` | smaps COW 取证 | 无 libart inline hook（LSPlant entry_point 替换） | ☐ |
| 匿名 rwxp/r-xp 页 | maps 权限扫描 | LSPlant trampoline 落点待查（§5 最后一项） | ☐ |
| `dlopen("libzygisk.so", RTLD_NOLOAD)` | 库驻留探测 | ZygiskNext 层负责，非本框架范围 | ☐ |
| `/proc/[pid]/cmdline` 含 lspd/zygiskd | 全系统进程扫描 | 守护进程命名 + Shamiko 挂载隐藏 | ☐ |
| 已安装应用列表含 Manager | PackageManager 枚举 | HMA | ☐ |
| Play Integrity verdict | Google PI API | PIF + keybox（海外 App） | ☐ |
| 壳 so CRC / guard 网络 | 企业版加固 RASP | 只 hook Java 层 + 中和检测消费点 | ☐ |
| 三相退出 | killProcess+exit+halt 并发 | hook 调用判定处而非退出原语 | ☐ |

## 7. 附录 B：文档推进路线

- [x] v0.1 检测面全景 + 本仓库消迹机制源码定位（2026-09-09）
- [ ] v0.2 真机基线：原版 1.9.2 vs 本版在 Hunter/Native Detector 的命中项差异表
- [ ] v0.3 LSPlant trampoline 内存落点分析（匿名页扫描面）
- [ ] v0.4 企业版加固专项：乐固/爱加密壳的 RASP 初始化时序与 hook 窗口
- [ ] v0.5 海外加固专项：DexProtector/Promon 行为检测特征库
- [ ] v1.0 输出可复用的「过加固检查清单」模块开发规范

---

## 8. 关键参考

- 本仓库源码：`daemon/src/main/jni/obfuscation.cpp`、`native/src/jni/hook_bridge.cpp`、`zygisk/src/main/cpp/module.cpp`
- LSPlant 方法 hook 流程：`external/lsplant`（entry_point 替换 + 动态 dex 代理，无 inline patch）
- 检测器实现参考：Native Detector / Hunter / Ruru / Duck Detector（LRFP-Team/LRFP 汇总）
- 同路线改版：JingMatrix/LSPosed（本仓库上游）、LSPosed-Irena、ijirak/CPosed、mywalkb/LSPosed_mod
- 分层对抗模块范例：`modules.lsposed.org/module/io.github.xalsace.qqbypass`

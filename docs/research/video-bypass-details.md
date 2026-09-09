# 「一只勇敢的小佳」三个视频 — 绕过技术细节整理

> 来源：UP 主「一只勇敢的小佳」（B 站 mid 1886803835）
> 获取方式：yt-dlp 下载音频 + faster-whisper（small）本地 ASR 转写，人工校正术语
> ⚠️ ASR 转写有识别误差，存疑处用 `(?)` 标注；详细实现作者放在其知识星球（付费），视频只讲思路
> 转写原文存档：`.workbuddy/tmp/videos/*.transcript.txt`（不入 git）

---

## 0. 术语对照表（ASR 误识别 → 正确术语）

| 转写原文 | 实际含义 |
|---|---|
| 霍克 / 获可 / hawk / 或K | hook |
| 无痕霍克 / 温霍格 / WinHawk | 无痕 hook（traceless hook） |
| nibret / NibrT / NIBART / 你比RT | libart(.so) |
| Freder / Fread / Fretrace / flat / Fractor | Frida（flat sour = frida-server） |
| Xbox | Xposed |
| LSP / LSPOST / LSPOS / suppose | LSPosed |
| 液表 / 孽表 | 页表（page table） |
| 可支形段 / 可真断 | 可执行段（r-xp） |
| Motin4 / Motus | /proc/self/mountinfo（挂载信息） |
| Mot Start | /proc/self/mountstats |
| WMA | VMA（vm_area_struct 内存区域链表/红黑树） |
| so nest | solist（linker 维护的已加载 so 链表） |
| rdbarg | r_debug（内含 link_map 链表） |
| GID Cache | jit-cache（Frida 伪装的 memfd 命名）(?) |
| 现成 | 线程（thread） |
| Pretrace | ptrace |
| 堆杖 | 堆栈（stack） |
| 夹袜层 | Java 层 |
| 测性道 / 刺激道 | 侧信道（side channel，此处特指时序侧信道） |
| trascade / trashkate | truncate() 系统调用（APatch 的 supercall 通道） |
| max | Magisk；aparch / Eparg / Epaqi / erparch = APatch；ksu = KernelSU |
| 牛头 | 牛头人检测（Native Detector 类检测器） |
| luhino / nohino / LowHigh Low | Nohello（闭源 KPM 隐藏模块）(?) |
| TX Store | TrickyStore（keybox 证书链模块） |
| KBOX | keybox（含公钥+私钥的设备密钥对） |
| talking | token（Play Integrity token） |
| DX | dex；QuickQuote = quick_code（ArtMethod 的 entry_point） |
| Night 5 | native（方法 native 化） |
| OBF / 动态挥销 | obfuscation 动态混淆 |
| Zegas / zxcode | Zygisk |
| Laop 文件 | loop 设备文件 |
| 多比 | Dobby（inline hook 库） |
| 星球 | 知识星球（作者的付费社群，详细实现代码在其中） |

---

## 1. 视频一 BV1XHnEzkEPL —「魔改 LSPosed 过海外加固」（2025-09-27，12 分钟）

### 核心改动
1. **对 LSPosed 代码做了修改，使特征进一步减少**；
2. **把 libart 上的 hook（trampoline）全部去除**——原话：「正常用 LSP 去 hook，libart 会产生一个分段问题；最新这一版把这个已经全部去除了，并且不影响正常功能」；
   - 「分段问题」= 视频三详解的 COW 分段：inline hook 写 r-xp 页触发写时复制，内核不再合并 VMA，同一 so 的可执行段在 maps 里拆成多段；
   - 去除方式：不用 inline hook 改 libart 代码，Java hook 全部走 ArtMethod 指针替换（与本仓库 LSPlant 路线一致）；
3. 所有 hook 都没有 libart 痕迹，稳定性高，**基本功能与用法和原版完全一致**（模块生态兼容）。

### 实测通过情况
| 目标 | 结果 | 备注 |
|---|---|---|
| Hunter | ✅ 直接过 | 残留报错仅为 APatch 挂载痕迹 + 代理检测（作者未处理代理 hook） |
| appdemo 加固（APKS 包） | ✅ | 用 MT 管理器确认壳后启动正常 |
| DexProtector（"低沙飘/DX飘"） | ✅ | 顺利 |
| scb 类加固（多个 App） | ✅ | 其中一个会**检测出 libart** → 处理：把相关应用勾进隐藏列表/改包名后通过 |
| kbiz / kplus（银行类(?)） | ⚠️ 大部分解决 | 新版加了 "VLS" 相关检测（VLS 待考），能进主页面但有残留检测点；只 hook 了系统属性，未 hook 壳本身 |
| wgd = Guardsquare/DexGuard | ✅ | 需在隐藏列表里勾选相关应用后通过 |
| vast（最新出现的加固） | ❌ 还有问题 | 作者明确说「vast 会比较麻烦一点」 |
| SVM 最新版(?) | ✅ 基本没问题 | |

### 海外加固厂商格局（作者口径）
「海外最主要四大加固厂商」+ 两个新晋：
- 四大：**STVD(?)**、**DexProtector**、**Appdome**、**Guardsquare(DexGuard)**
- 新晋：oneplus(?)（见得少未演示）、**vast**（最麻烦，目前未完全过）

### 实操要点
- 处理残留检测的通用手法：**把目标 App 及相关组件勾选进隐藏应用列表（HMA）/作用域**；
- 个别顽固检测「把应用删掉改个包名（重装）」也能过，说明部分检测锚定包名/签名。

---

## 2. 视频二 BV1ipdpBwEeh —「内核模块 + 无痕 hook + 最强 LSP」（2026-04-18，12 分钟）

### 2.1 内核模块的「读写页表切换」无痕 hook（重点）

**原理**：内核模块维护两套页表——
- 代码**执行**时切到「hook 页表」（页面里是 patch 后的指令）；
- 内存**读取**时切到「原始页表」（页面里是未修改的原始指令）。

检测器（如 Hunter 的 CRC 校验）读到的永远是原始指令 → **prologue 校验 / .text 完整性校验全部失效**。

**原版 Frida 直接跑在该机制上会死循环的原因**（作者详细讲解）：
- Frida 跳板要占满目标函数**前四条指令**（`LDR X16, #off; BR X16` 形式），跳板的 LDR 需要从当前 PC 页**读取**函数地址；
- PC 与被保护页在**同一页**：页表切到「可执行」就没有读权限、切到「可读」就没有执行权限 → PC 永远读不到自己的跳板数据 → 死循环/崩溃；
- 作者做了优化（未公开细节，在星球），使**原版 Frida 全 API** 也能跑在页表切换机制上；之前还有个 Rust 版 Frida（API 不全，效果欠佳）。

**实测演示**：
- 普通方式 hook libart 的 dex 解析函数 → Hunter 的 CRC 检测报 **libart 被修改**；
- 换无痕 hook → Hunter 全绿；hexdump 查看函数头是**原始指令**（无改动）；`/proc/self/maps` 里 libart **无分段**（无 COW 拆分）；
- 使用方式：JS 脚本先调一个对外提供的初始化函数（与内核通讯），之后 hook 代码写法与 Frida API 一致。

### 2.2 Frida Gadget 注入小工具
- 功能：**选进程**注入（多进程 App 可指定子进程）、写 Gadget config（JSON，照 Frida 官网 gadget config 文档写，放任意目录如 /data/local/tmp）、默认端口 `14x5x`(?)（可自改），端口转发后即可用原版 Frida 连接；
- 内核模块对挂载等做了深度隐藏，Hunter、牛头人均全绿。

### 2.3 魔改 LSP（与视频一呼应）
- 演示爱加密（ijiami）加固的 App：模块正常加载（logcat 可见 hook 日志），App 无任何异常；
- **兼容绝大部分 LSPosed 模块**（HMA 隐藏应用列表等系统级模块正常）；
- 支持 Android 8–15；Android 16 待适配（多了 16K 页表问题）；
- 关键设置：**ZygiskNext 用 1.3.2 或 1.3.4**，并打开「匿名内存」和「连接器」两个选项（不开会出问题）；
- 个别 App 需在管理器里打开「排除模块」选项。

---

## 3. 视频三 BV1WEovBjEVh —「检测总览」（2026-04-26，39 分钟，最硬核）

> 全视频主线：**检测的本质是检测异常环境**。要对抗必须先分清：是 root 被检测、注入被检测、还是 hook 被检测——逐层卸载可疑点、逐个排查。

### 3.1 Root / 解锁层检测（无注入时也存在）
| 检测点 | 细节 |
|---|---|
| prop 属性 | 解锁后系统属性变化（如 `ro.boot.flash.locked`、`ro.boot.verifiedbootstate` green/orange、`ro.debuggable` 等），可判定解锁 |
| 硬件证明（Key Attestation） | Google KeyStore/TEE 生成证书链，携带是否解锁、APK 签名等信息 → **TrickyStore** 模块可伪造完整证书链，但需要 **keybox**（公钥+私钥）；attestation 过了 Play Integrity token 也就过了 |
| 杂项 | 系统更新日期过期、是否 Pixel 等高风险机型名单 |
| su 路径 | `/system` 下 su 文件（可用排除方式去除） |
| **挂载痕迹** | `/proc/mounts`、`/proc/self/mountinfo`、`/proc/self/mountstats` 三个文件都能看；Magisk/APatch 的模块挂载条目清晰可见 |
| SELinux | 遍历全部 fd 的扩展 SELinux 属性可见 Magisk 相关内容 |
| init 文件 | Magisk root 原理是替换 init → 检测 init 是否被替换/含 Magisk 特征 |

**mountinfo 连续性检测（牛头人等的精准手法）**：
- mountinfo 每行有 **mount ID 和父 mount ID**；正常系统的 ID 序列有连续规律；
- 隐藏模块的常规做法是内核里过滤这两行（不展示）→ **ID 序列断档** → 被精准定位「此处应有连续条目」；
- 正确处理的难点：隐藏两行后**下面所有 mount ID 都要跟着变**，且父 ID 必须真实存在、全局唯一——配套修改极复杂；
- **Nohello**(?)（闭源 KPM 模块）针对此做了较多处理；LSPosed 自身也有挂载痕迹（mountinfo 里出现 zygisk 相关条目）；
- 作者给出的原则：**保持部分片段连续（而非整体连续）+ 保证全局父 ID 唯一存在**；
- APatch 早期有 loop 设备文件痕迹（`/dev/block/loop*`），新版已无此问题。

### 3.2 时序侧信道检测（root 管理器通讯通道）
- APatch/KSU 与内核通讯走**系统调用**：APatch 的 supercall 用的是 **truncate()**——path 带超级密钥 key，length 为魔数；
  - length=0：无 hash 计算，直接走正常 truncate（快）；
  - length=魔数：内核做 hash 计算与预存 key（MD5 等保存）比对（**必然更慢**）；
  - 检测方以正常调用为**参照物**做时序对比 → 有 key 的路径耗时异常长 → 判定 APatch 存在；
- KernelSU：hook 了 `statx` 系某个方法但没 hook 底层同逻辑的另一个方法 → 两个本应耗时接近的 API **时间差异常大** → 判定 KSU；
- 对抗：Nohello(?) 对 truncate 类侧信道做了处理（闭源未完全逆向，作者只摸清大概）；用模块方式处理时序侧信道很麻烦。

### 3.3 注入检测（核心层）
**前提**：先把 root 基准环境清理干净，才轮到注入层。

| 检测手段 | 细节 | 对抗 |
|---|---|---|
| **匿名可执行段** | 正常应用除自身 so 外不应有可执行段；maps 里出现**无名字的 r-xp 段**（除 vdso/anon 系统段外）= 必有问题。最狠的一招 | 需模块级隐藏 |
| **maps / smaps / map_files 三通道** | VMA 信息有三个 /proc 展示口：只处理 maps 不处理 **smaps**（含 Private_Dirty 等 COW 证据）或 **/proc/self/map_files** 都会被抓 | 三个通道必须同时处理 |
| **solist** | linker 维护已加载 so 链表，先 mmap 一大块内存再逐个分配 soinfo；可遍历查出全部加载的 so | 从 solist 摘除 |
| **r_debug → link_map** | r_debug 内部还有 link_map 链表；**只摘 solist 不摘 link_map 一样被检测**；但摘 link_map 极端情况下会导致应用**死锁**（dlopen 持锁遍历） | 需配套处理 |
| **fd 排查** | frida-server 与前端通讯的 socket fd、SELinux 属性、`readlink(/proc/self/fd/*)` 查路径；Zygisk/memfd 匿名段也有打开的 fd | fd 级别隐藏 |
| **jit-cache 伪装识别** | Frida 把 memfd 改名为 jit-cache 伪装成 ART 的 JIT 缓存；但系统**正常只存在一个** jit cache，**存在多个 = 有问题**；可用 readlink 查出 | 命名唯一性 |
| **内存特征字符串** | 四大途径：**maps / smaps / map_files / solist**，查 frida/xposed/跳板特征字节等字符串，匹配任一即判定 | 特征字符串随机化 |
| **dex 排查** | 遍历全部 ClassLoader 查所有 dex 里有无 LSPosed 字样；查是否存在 LSPosed 的 ClassLoader | dex 混淆（见 3.5） |

### 3.4 hook 检测
| 检测点 | 原理 | 对抗 |
|---|---|---|
| **inline hook 权限痕迹** | r-xp 段要写入须先 mprotect 成 rwxp；**Frida 不改回去** → maps 里可见 rwxp 段 | Dobby 会改回，但引出下条 |
| **COW 分段** | 写 r-xp 页触发写时复制 → 该页物理上与 zygote 母页分裂 → 内核不再合并 VMA → **同一 so 的可执行段拆成多段**（正常 libc.so 可执行段只有一段，hook 后变多段）→ Hunter 直接可查 | 页表切换（视频二）/ 不用 inline hook |
| **指定函数检查** | 检测方明确知道目标函数时，直接查该函数头指令/是否被 native 化，无法防御正面硬刚 | **在函数的上下游做操作**（hook 调用方/被调方替代直接 hook 目标） |
| **Frida Java hook 的 native 化** | Frida 的 Java hook 是 JS↔native 交互，会把目标 Java 方法**变成 native 方法** → 直接查方法 flags 是否 native 即抓 | 换实现 |
| **LSPosed 的 entry_point 指针** | ART 方法的 quick_code 跳板不再指向 libart.so，而是指向 **LSPosed 的内存地址** → 可检测；但检测方必须**明确知道 hook 的是哪个函数**才能查 | 同上：上下游 hook；每个方法对应一个 ArtMethod，逐个查成本高 |
| **堆栈字眼** | 堆栈里查 XposedBridge/LSPosed/frida 相关类名字眼 | JNI 转发不留 Java 帧 |
| **线程名** | Frida 的 gdbus/gmain/gum-js-loop 等线程名，遍历 `/proc/self/task/*/comm` 即抓；**不必改 Frida 源码**（改源码名字固定，风控后台反而可收录），可在**用户层动态改名** | 用户层改线程名 |
| **ptrace** | 被 ptrace 的进程别人 attach 不上 → fork 子进程 ptrace 自己试探；或 ptrace 后向外发信号验证。**建议不要直接用 ptrace**；Zygisk/Frida 注入 zygote 也用 ptrace，但注入后立刻卸下 ptrace 所以无特征 | 用 Zygisk 系（1.3.2/1.3.4） |
| **Java 层默认 hook** | LSPosed/Frida **默认 hook 了一些 Java 层函数**，这些函数本身被检测方直接查也能查到 | 需自行处理默认 hook 点 |
| binder 相关 | 提及未展开 | — |

### 3.5 字符串/类混淆（OBF）
- LSPosed 已内置一些 **obfuscation 规则**（动态混淆，每次对 LSPosed 相关类做混淆）——对应本仓库 `daemon/src/main/jni/obfuscation.cpp` 的 dex 签名随机化；
- 但作者明确说：**「它做的混淆并不是完全足够的」**，想做到没有任何类被检测，需要自己针对这些类做更深的处理（第三方模块自定义特征类不在覆盖范围内）。

### 3.6 收尾结论（作者原话要义）
- 把上面这些点都解决后，市面上绝大部分检测都难以再判定；
- 但「检测对应的隐藏，并没有原理上说的那么好弄」，尤其在特殊环境（如 mountinfo 连续性 + loop 文件 + 侧信道要配套处理）；
- 详细实现代码在其知识星球。

---

## 4. 三视频串联出的完整对抗链（与我们研究的映射）

```
root 基准环境   → 挂载痕迹(mountinfo连续性)、SELinux、init、侧信道时序   [内核模块/Nohello 层]
注入层         → 匿名可执行段、maps/smaps/map_files 三通道、solist+link_map、fd [ZygiskNext+Shamiko 层]
hook 层        → 无 libart inline hook（去 trampoline）、entry_point 替换、   [魔改 LSP ≈ 本仓库 Vector 路线]
                  native 化检测、上下游 hook、堆栈/线程名消迹
Java/类层       → dex 签名随机化(OBF) + 第三方模块类自行深混淆              [obfuscation.cpp + 模块规范]
内存读取对抗    → 页表切换无痕 hook（读原始指令/执行 patched 指令）           [内核模块层，对抗 CRC/分段]
App 专项       → HMA 勾选、包名重装、排除模块开关                           [个案处理]
```

**与本仓库（JingMatrix/Vector）的对应**：视频一的「去 libart trampoline」≈ LSPlant entry_point 替换（§3.4）；视频三的「OBF 动态混淆」≈ `daemon/src/main/jni/obfuscation.cpp`（且作者确认其覆盖不足，印证 P3.2 课题价值）；「solist 查库」与本仓库 HEAD #959 方向一致（用 solist 而非 maps）。

**新增的研究课题**（回填执行计划）：
1. mountinfo 连续性：本框架 zygisk 挂载条目是否导致 mount ID 断档（P1 基线加测）；
2. smaps/map_files 两通道在本框架下是否泄露（附录 A 补测项）；
3. link_map 是否残留 liblspd 条目（Hunter 实测验证）;
4. 「上下游 hook」原则写入模块开发规范（P4）。

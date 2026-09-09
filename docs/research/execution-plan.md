# 反加固对抗研究 — 执行计划

> 分支：`docs/anti-hardening-research`（文档）/ `explore/anti-hardening-bypass`（代码实验）
> 上游基线：JingMatrix/LSPosed（Vector）master `efb82883`（canary-3111）
> 关联文档：`docs/research/anti-hardening-bypass.md`（检测面全景与消迹机制）
> 日期：2026-09-09

---

## 0. 环境现状（已实测）

| 项 | 状态 | 备注 |
|---|---|---|
| 当前分支 | `docs/anti-hardening-research` | 文档分支；代码实验切 `explore/anti-hardening-bypass` |
| Git 子模块 | ❌ 9 个全部未初始化 | LSPlant/LSPlt/Dobby/fmt/xz-embedded/libxposed×2/commons-lang/manifest-editor |
| JDK | ✅ 21.0.12（Temurin） | 新版 Vector 用 JDK 21 即可（不像 hsapiserver 需要 JDK 8 编译） |
| Android SDK | ✅ `~/Library/Android/sdk` | 含 platform-tools、build-tools |
| NDK | ✅ 29.0.14206865 | |
| local.properties | ❌ 不存在 | 需创建指向 SDK |
| adb 设备 | ❌ 当前无设备连接 | 真机阶段需接 Pixel 6 / OnePlus 9 Pro |
| Gradle 模块 | `:daemon :dex2oat :legacy :manager :manager-ui :services:* :xposed :zygisk` | rootProject = Vector |

---

## 1. 目标

1. 验证 Vector 版 LSPosed 的反检测机制在真机上的实际效果（对照 `anti-hardening-bypass.md` 附录 A 核查表逐项打勾）；
2. 产出「原版 LSPosed vs Vector 版」检测命中差异数据；
3. 实测对红果/抖音等加固 App 的 hook 可行性（衔接 hongguo capture / douyin 工作区）；
4. 最终沉淀可复用的「过加固检查清单 + 模块开发规范」（文档 v1.0）。

## 2. 阶段计划

### P0 — 构建环境就绪（0.5 天，本机即可完成）
| # | 任务 | 验收标准 |
|---|---|---|
| 0.1 | `git submodule update --init --recursive`（9 个子模块） | `git submodule status` 无 `-` 前缀 |
| 0.2 | 创建 `local.properties` 指向 `~/Library/Android/sdk` | gradle 不再报 SDK 缺失 |
| 0.3 | `./gradlew :zygisk:assembleRelease :manager:assembleRelease` | 产出 zygisk 模块 zip 与 manager apk |
| 0.4 | 记录构建产物哈希存档 | `docs/research/builds/` 记录 sha256 + commit |

**风险**：子模块走 HTTPS 需过代理（本机代理 127.0.0.1:65353，git 已验证 SSH 可用；HTTPS 502 需切代理或改 SSH）。若卡壳 → 用 `git config url."git@github.com:".insteadOf "https://github.com/"` 兜底。

### P1 — 真机基线检测（1 天，需 Pixel 6 或 OnePlus 9 Pro）
| # | 任务 | 验收标准 |
|---|---|---|
| 1.1 | 刷入 Vector 构建（KernelSU/Magisk + ZygiskNext 环境） | Manager 显示框架激活 |
| 1.2 | 安装检测器：Native Detector / Hunter / Ruru / Momo | 四个工具跑完并截图 |
| 1.3 | 对照附录 A 核查表逐项记录命中情况 | 核查表 11 项全部有结论（过/不过/N.A.） |
| 1.4 | 原版 LSPosed 1.9.2 同机重刷，跑同一套检测 | 产出差异对比表 |

**产出**：`anti-hardening-bypass.md` 附录 A 的 ☐ 全部打勾 + v0.2 差异表。

### P2 — 加固 App 实测（2 天，衔接现有逆向工作区）
| # | 任务 | 验收标准 |
|---|---|---|
| 2.1 | 红果（hongguo）：在 Vector 框架下加载既有 SSL pinning bypass 模块 | 壳未闪退、未触发风控登出，抓包链路通 |
| 2.2 | 抖音（com.ss.android.ugc.aweme）：同法验证 | 同上 |
| 2.3 | 对比原版框架下同一模块的触发情况 | 明确 Vector 是否降低了壳检测触发率 |
| 2.4 | 若触发：定位具体检测点（日志/smaps/maps 快照） | 输出触发点清单，回填附录 A |

**原则**：模块代码不动逻辑，只换框架环境做 A/B；调试输出不进 git。

### P3 — 专项深挖（各 1 天，可与 P2 并行）
| # | 课题 | 关键问题 |
|---|---|---|
| 3.1 | LSPlant trampoline 内存落点 | 代理 dex 的 trampoline 是否落在匿名可执行页？能否挪入已有 RX 段？（附录 A 第 5 项） |
| 3.2 | obfuscation 覆盖面 | 第三方模块 dex 中的自定义特征类是否也被随机化？漏网面有多大？ |
| 3.3 | 企业版加固时序 | 乐固/爱加密壳 RASP 初始化 vs `handleLoadPackage` 回调的先后窗口 |

### P4 — 沉淀输出（0.5 天）
- `anti-hardening-bypass.md` 升到 v1.0：附录 A 全部实测结论 + 「过加固模块开发规范」（hook 点选择原则、禁做清单、检测消费点中和模式）；
- 视复用价值，把「分层对抗环境搭建流程」存为 skill。

## 3. 约束与红线

1. **分支隔离**：文档只进 `docs/anti-hardening-research`；任何代码改动只在 `explore/anti-hardening-bypass` 开改，且先开分支再动；
2. 平行 session 不交叉感染；临时调试/测试输出不提交 git；
3. 已有逻辑不删不改，URL 参数原样使用（涉及红果任务时）；
4. 真机刷写前备份 boot/init_boot 镜像，变砖可回退；
5. 对资源敏感的操作（轮询、日志抓取）限定次数与时长，参考既往约定（重试上限 5 次/请求）。

## 4. 依赖与阻塞项

| 依赖 | 影响阶段 | 状态 |
|---|---|---|
| 子模块可拉取（网络/代理） | P0 | ⚠️ HTTPS 曾 502，需试 SSH insteadOf |
| Pixel 6 / OnePlus 9 Pro 可接线 | P1/P2 | ❌ 当前 adb 无设备 |
| 设备已有 KernelSU + ZygiskNext | P1 | 待确认 |
| 检测器 APK（Hunter 等）获取 | P1 | LRFP-Team/LRFP 仓库汇总 |
| 红果/抖音 hook 模块现成可用 | P2 | ✅ hongguo capture / douyin 工作区已有 |

## 5. 完成定义（DoD）

- [ ] 附录 A 核查表 11 项全部有真机结论；
- [ ] 原版 vs Vector 检测差异表产出；
- [ ] 红果 + 抖音各至少一次完整「hook 成功且壳未触发」记录；
- [ ] 文档 v1.0 合入并推送。

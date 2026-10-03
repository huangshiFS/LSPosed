# 构建产物记录

基线 commit: `fb649172` | 日期: 2026-10-03 | Gradle 9.7.0 / JDK 21 / NDK 29.0.14206865 / CMake 3.31.6

| 产物 | 大小 | sha256 |
|---|---|---|
| `zygisk/build/outputs/apk/release/zygisk-release-unsigned.apk` | 5.6 MB | `07232c8c74b7900e2350da6a67d7965a33070414966ed077bae97befa0729ba7` |
| `manager/build/outputs/apk/release/manager-release.apk` | 3.1 MB | `d048afc1b8c46fdb3578930e4059d4a160a954ede4622ec9582301b9980b466c` |

构建命令：
```bash
./gradlew :zygisk:assembleRelease :manager:assembleRelease
```

注意：zygisk 产物为未签名 APK（模块内嵌 so，需由刷机脚本打包成 Magisk/KernelSU 模块 zip 后刷入）。

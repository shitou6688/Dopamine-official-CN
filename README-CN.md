# Dopamine（官方 3.0.13）国内网络构建版

基于 **官方 [opa334/Dopamine](https://github.com/opa334/Dopamine) 的 `3.0.13` release tag**，通过 GitHub Actions 的 macOS runner 编译打包。目标：**任何支持越狱的设备，用普通签名安装后，不挂代理也能下载内核**。

> 与 `Dopamine-roothide-CN` 是两条独立产线。本仓库针对官方主线（rootless），不含 roothide 改动。

## 为什么官方版会卡在下载内核

官方 `Application/Dopamine/Dependencies/libgrabkernel2.a` 里硬编码了两个境外元数据接口：

```
https://api.appledb.dev/ios/main.json.xz      （约 72 MB）
https://api.appledb.dev/ios/%@;%@.json
```

每次越狱时 `grab_images()` 先拉这份索引才能确定 IPSW 直链，国内普通网络拉不动。但固件本体在 `updates.cdn-apple.com`，国内直连有边缘缓存节点，实测返回 `206 Partial Content`、`X-Cache: HIT`，20 MB 内核片段 2–4 秒取完。

本仓库做的就是把「解析元数据」这一步从**设备端**挪到**构建机**：CI 在打包时读取所有 IPSW 的 ZIP 中央目录（只取文件名 / offset / length / 压缩方式 / 原始大小 / CRC32，不下载文件内容），生成 `KernelIndex.plist` 内置进 IPA。设备端用 `hw.machine|kern.osversion` 查表，只向 Apple CDN 发 3 个 Range 请求，取回后 raw-inflate 并用 CRC32 验真，再落到 `Documents/kernelcache`、`sptm.im4p`、`txm.im4p`。查不到条目才回落原 `grab_images()`。

## 产物

| 产物 | 设备端网络行为 | 适用 |
| --- | --- | --- |
| `Dopamine-CN-3.0.13-<hash>.ipa` / `.tipa` | 只向 `updates.cdn-apple.com` 发 3 个 HTTP Range（约 20 MB），**不访问 appledb.dev，不需要代理** | 通用包，覆盖所有支持越狱的机型/系统 |
| `Dopamine-CN-...-OFFLINE-<机型>-<Build>.ipa` / `.tipa` | **零联网下载**，内核三件套已内置 | 仅该机型 + 该 Build，不能跨设备 |

`coverage=all`（默认）内置全量索引：5,468 条 `<机型>|<Build>`、181 个设备标识、索引约 4.89 MB，覆盖上游 `-versionSupportString` 声明的 iOS/iPadOS 15.0–18.7.1 与 A12/A13 的 26.0–26.0.1。

`.ipa` 给普通签名/侧载，`.tipa` 给 TrollStore。两者内容一致，仅扩展名不同。（TrollStore 安装时上游本来就直接读系统 kernelcache，不存在下载问题。）

## 三步使用

1. 把本目录的 `.github/` 和 `Tools/` 放进一个 GitHub 仓库（空仓库即可，工作流会自动克隆官方 `3.0.13` tag），或 fork 官方仓库后加进去。
2. Actions → `Build Dopamine (official) CN - no proxy on device` → Run workflow。默认 `mode=index` + `coverage=all` 就是全设备免代理版，直接跑即可。
3. 在 Artifacts 下载 `Dopamine-official-CN-all`，或勾选 `make_release` 发到 Releases 方便手机直接下载。

只给自己一台机做完全离线包时：把 `coverage` 改 `single`、`mode` 改 `both`，并在 `Tools/targets.txt` 写 `机型|Build`（机型用 `ideviceinfo -k ProductType`，Build 在设置 → 通用 → 关于本机）。也可直接在 Run workflow 里填 `device` + `os_build`。

## 与 roothide 版工作流的构建链差异

1. **官方仓库不内置 bootstrap**。`project.pbxproj` 把 `bootstrap_1800.tar.zst` / `bootstrap_1900.tar.zst` 列为 bundle 资源，但 git 里没有，官方 CI 用 `Resources/download_bootstraps.sh` 从 `apt.procurs.us` 拉取（各约 20 MB）。本工作流把这一步做成带重试的显式下载，放在 `xcodebuild` 之前；**只在 runner 上跑，与设备端网络无关**。roothide fork 把这两个包直接提交在仓库里，所以那版不需要。

2. **BaseBin 守护进程需要补 `-lz`**。官方 `BaseBin/dopamine/Makefile` 把 `Application/Dopamine/Jailbreak/*.m` 一起编译进 `dopamine` 守护进程，但它的 `LDFLAGS` 只有 `-lcompression`，没有 zlib。补丁头里的 `crc32()` / `inflate()` 因此链接报 `Undefined symbols for architecture arm64`。工作流在打完补丁后给该 Makefile 的 `LDFLAGS` 加 `-lz`（幂等）。iOS app target 在 `project.pbxproj` 里本来就链接了 `libz.tbd`，不受影响。

其余复用同一套方案：THEOS（官方 `theos/theos` 安装脚本 + iPhoneOS16.5 SDK）、Procursus、trustcache、`gmake NIGHTLY=1`。工具链固定 `macos-15` + Xcode 16.4 —— Xcode 26 / Apple clang 21 链接 `watchdoghook.dylib` 会段错误，这是 CI 实测排出来的。

## 源码补丁

只动 `Application/Dopamine/Jailbreak/DOEnvironmentManager.m` 的 4 处，幂等，重复执行不叠加：

1. 加一行 `#import "DO_CN_KernelSource.h"`；
2. `-accessibleKernelPath` 的 `grab_images()` 下载块换成「先查内置索引，失败回落」；
3. `-accessibleSPTMPath` / `-accessibleTXMPath` 额外识别 bundle 里的 `sptm.im4p` / `txm.im4p`（与 `grab_images` 写出的格式一致）。

设备端实现全部内联在 `Tools/DO_CN_KernelSource.h`，**不改 `.xcodeproj`** —— 官方工程本来就链接了 `libz.tbd` / `libcompression.tbd`。补丁锚点在官方 3.0.13 与 roothide 3.x 中逐字相同。

## 文件

| 路径 | 作用 |
| --- | --- |
| `.github/workflows/build-cn.yml` | macOS runner 全流程：克隆官方 tag + 子模块 → THEOS/Procursus/trustcache → 下载 bootstrap → 打补丁 → 全量索引 → 编译 → 重签名重打包 → Artifact / Release |
| `Tools/cn_kernel.py` | 远程 ZIP64 读取，`selftest / probe / index / fullindex / fetch / patch / pack` 七个子命令，纯标准库 |
| `Tools/DO_CN_KernelSource.h` | 设备端运行时，全内联 |
| `Tools/targets.txt` | `coverage=single` 时的目标清单 |

## 验证状态与限制

**CI 全量构建已跑通**：run [`42829205538`](https://github.com/shitou6688/Dopamine-official-CN/actions/runs/42829205538)，18 个步骤全绿（21 分 56 秒），Release [`cn-3.0.13-2`](https://github.com/shitou6688/Dopamine-official-CN/releases/tag/cn-3.0.13-2) 已发布 2 个资产：

| 文件 | 大小 |
| --- | --- |
| `Dopamine-CN-3.0.13-5b1d2c1.ipa` | 55,902,457 B |
| `Dopamine-CN-3.0.13-5b1d2c1.tipa` | 55,902,457 B |

已从产物中解包核对：`KernelIndex.plist` 为 schema 2、5,468 条、181 设备、0 跳过（4,892,462 B）；`bootstrap_1800.tar.zst`（20,792,571 B）、`bootstrap_1900.tar.zst`（20,772,810 B）、`sileo.deb`、`zebra.deb` 均在 bundle 内；二进制含 `Downloading Kernel (CN direct)` 补丁标记。

国内直连实测（显式绕过本机应用层代理）：用产物索引里的 `iPhone15,3|23A355`（iOS 26.0.1）向 Apple CDN 发 3 个 Range，全部 `HTTP 206`，kernel 19,901,404 B / 2.59 s，sptm / txm 各 171,313 / 160,451 B，CRC32 三条全对。

- 已验证：官方 3.0.13 源码上补丁的精确 diff 与幂等性、ZIP64 目录解析、真实 IPSW 内核的 CRC32 端到端一致性、工作流 YAML 与脚本语法、CI 全量编译 + 重签名打包 + Artifact/Release 上传。
- 二进制里仍能搜到 `api.appledb.dev` 字符串，来自静态链接的 `libgrabkernel2.a` 回落路径；命中索引时设备端不会发起该请求。
- **未执行**：真机越狱验证（需要实际设备）。
- 索引按 `<机型>|<Build>` 固化，Apple 发布新系统后需重跑工作流刷新；未命中会静默回落原 `grab_images()`，功能正常但那台设备仍需代理。
- 旧版本 IPSW 若被 Apple 下架导致直链 404，构建期自动跳过该条，或用 `firmware_url` 指定可访问的直链。
- 通用包不是零流量，仍需约 20 MB 下载；要零联网只能用单机离线包。

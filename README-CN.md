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

## 与 roothide 版工作流的唯一实质差异

**官方仓库不内置 bootstrap**。`Application/Dopamine.xcodeproj/project.pbxproj` 把 `bootstrap_1800.tar.zst` / `bootstrap_1900.tar.zst` 列为 bundle 资源，但 git 里没有，官方 CI 用 `Resources/download_bootstraps.sh` 从 `apt.procurs.us` 拉取（各约 20 MB）。本工作流把这一步做成了带重试的显式下载，放在 `xcodebuild` 之前；**这一步只在 runner 上跑，和设备端网络无关**。roothide fork 把这两个包直接提交在仓库里，所以那版不需要。

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

- 已验证：官方 3.0.13 源码上补丁的精确 diff 与幂等性、ZIP64 目录解析、Apple CDN 的 206 Range、真实 IPSW 内核的 CRC32 端到端一致性、工作流 YAML 与脚本语法。
- CI 全量编译结果：见仓库 Actions 与 Releases（跑完后在此补充 run 号）。
- **未执行**：真机越狱验证（需要实际设备）。
- 索引按 `<机型>|<Build>` 固化，Apple 发布新系统后需重跑工作流刷新；未命中会静默回落原 `grab_images()`，功能正常但那台设备仍需代理。
- 旧版本 IPSW 若被 Apple 下架导致直链 404，构建期自动跳过该条，或用 `firmware_url` 指定可访问的直链。
- 通用包不是零流量，仍需约 20 MB 下载；要零联网只能用单机离线包。

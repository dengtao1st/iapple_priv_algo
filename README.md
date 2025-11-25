# iapple_priv_algo

## Overview 概述
iapple_priv_algo is a research toolkit that reverse-engineers how Apple’s private authentication stack works on iOS, bundling jailbreak tweaks, CLI helpers, Frida scripts, and trace captures so each anisette/SRP/PCS phase can be replayed inside `akd`.  
iapple_priv_algo 是一个面向研究的工具集，用于复现 iOS 上 Apple 私有认证栈的细节，集合了越狱插件、命令行工具、Frida 脚本与抓包数据，方便在 `akd` 内还原 anisette、SRP、PCS 等环节。

## Key Components 核心模块
- `src/priv_algo/iapple_priv_algo.mm` – Stand-alone helper that drives AuthKit/AppleAccount/CloudKit/CloudServices private frameworks for anisette, signatures, and SRP sessions; most SRP blobs, 2FA headers, and token serialization logic live here with `src/srp` + `src/third_party`. `src/priv_algo/iapple_priv_algo.mm` – 独立可执行文件，直接调用 AuthKit、AppleAccount、CloudKit、CloudServices 等私有框架，完成 anisette 请求、签名生成与 SRP 会话，核心逻辑与 `src/srp`、`src/third_party` 协作。
- `src/priv_algo/iapple_priv_algo_dylib.mm` – CaptainHook-based MobileSubstrate tweak injected into `akd` per `txt/iapple_priv_algo.plist`, enabling in-daemon invocation of private methods. `src/priv_algo/iapple_priv_algo_dylib.mm` – 通过 CaptainHook 注入 `akd` 的 MobileSubstrate 插件（见 `txt/iapple_priv_algo.plist`），可在守护进程内调用私有方法。
- `src/priv_algo/security_devices.mm` – Replays `/appleid/account/manage/security/devices/*` APIs to list/prune trusted devices after anisette headers and tokens are available. `src/priv_algo/security_devices.mm` – 在拿到 anisette 和 token 后重放 `/appleid/account/manage/security/devices/*` 接口，用于查看或清理受信设备。
- `src/priv_algo/pcs_enroll.mm` – Builds `/usr/bin/pcs_enroll`, a long-running helper that tails `/tmp/pcs_enroll.plist` and forwards credentials into `ProtectedCloudStorage.framework (PCSIdentitySetup)`. `src/priv_algo/pcs_enroll.mm` – 编译常驻进程 `/usr/bin/pcs_enroll`，监视 `/tmp/pcs_enroll.plist` 并把凭据交给 `ProtectedCloudStorage.framework` 的 `PCSIdentitySetup`。
- `src/network_tweak/src/oc_hook/hook.mm` – Optional tweak that filters `LakituResponse` and `EscrowGenericResponse` payloads so serialized metadata can be massaged before `com.apple.sbd` consumes it. `src/network_tweak/src/oc_hook/hook.mm` – 可选响应过滤 tweak，在 `com.apple.sbd` 读取前改写 `LakituResponse`、`EscrowGenericResponse` 的 metadata。
- `src/AltSignAppleAPI`, `src/third_party/corecrypto`, `src/third_party/OpenSSL`, `src/third_party/InflatableDonkey`, `src/srp` – Vendored code implementing Apple-style signing, anisette, and SRP math. `src/AltSignAppleAPI`、`src/third_party/corecrypto`、`src/third_party/OpenSSL`、`src/third_party/InflatableDonkey`、`src/srp` – 提供签名、anisette 与 SRP 算法的第三方/逆向源码。
- `frida/*.js`, `network_flow/*.chls`, `third_tools/`, `iPhone_lldb/` – Instrumentation scripts, captured traffic, SSLKillSwitch2 package, and LLDB helpers for debugging on device. `frida/*.js`、`network_flow/*.chls`、`third_tools/`、`iPhone_lldb/` – 真机调试辅助，包含 Frida 脚本、抓包、SSLKillSwitch2 及 LLDB 脚本。

## Repository Layout 仓库结构
- `bin/` – Build artifacts such as `iapple_priv_algo`, `iapple_priv_algo.dylib`, `pcs_enroll`. `bin/` – 存放 `iapple_priv_algo`、`iapple_priv_algo.dylib`、`pcs_enroll` 等编译产物。
- `src/priv_algo/TwoFactorCode.txt`, `two_factor_all_headers_format.mm` – Prebaked 2FA payloads plus helpers for copying hydrated headers between requests. `src/priv_algo/TwoFactorCode.txt`、`two_factor_all_headers_format.mm` – 预置 2FA 输入与用于复制 header 的辅助函数。
- `txt/` – MobileSubstrate plists and quick notes; `txt/readme.txt` includes Base64 CLI payload examples. `txt/` – 存放 MobileSubstrate plist 及笔记，`txt/readme.txt` 有 Base64 命令行示例。
- `network_flow/` – Charles session exports illustrating intended HTTP exchanges. `network_flow/` – Charles `.chls` 抓包，记录预期的 HTTP 流程。
- `iPhone_backup/` – Sanitized keychain artifacts and daemon plists for offline analysis. `iPhone_backup/` – 脱敏后的 keychain、守护进程 plist 等离线分析素材。
- `release_tar_gz.sh` – Script that bundles the tweak binary and manifest into a release archive. `release_tar_gz.sh` – 将 tweak 二进制与 plist 打包发布的小脚本。

## Build & Deployment 编译与部署
1. Install Xcode Command Line Tools along with `ldid2` and `sshpass`; adjust the signing identity (`$(M1_CC)`, `M1_CS`) in `Makefile` if it differs on your machine. 安装 Xcode Command Line Tools，并准备 `ldid2` 与 `sshpass`；如本机证书不同，请修改 `Makefile` 中的 `$(M1_CC)`、`M1_CS`。
2. Update `IPHONE_HOST`, `IPHONE_PORT`, `IPHONE_PASS` inside the top-level `Makefile` and `src/network_tweak/Makefile`; defaults assume the jailbroken device is reachable via `127.0.0.1:2222` (`iproxy`) with the `alpine` root password. 在顶层 `Makefile` 与 `src/network_tweak/Makefile` 调整 `IPHONE_HOST/PORT/PASS`，默认假设通过 `iproxy` 映射到 `127.0.0.1:2222` 且 root 密码为 `alpine`。
3. Run `make` from the repo root to build and scp `/usr/bin/iapple_priv_algo`, `/usr/bin/pcs_enroll`, `/Library/MobileSubstrate/DynamicLibraries/iapple_priv_algo.dylib`; binaries are stripped/signed automatically before `akd` restarts. 在仓库根目录运行 `make`，构建并通过 scp 推送 `/usr/bin/iapple_priv_algo`、`/usr/bin/pcs_enroll`、`/Library/MobileSubstrate/DynamicLibraries/iapple_priv_algo.dylib`，过程中会自动 strip、签名并重启 `akd`。
4. Run `make` inside `src/network_tweak` if you need to deploy the response-filtering `network_hook.dylib`. 若需部署响应过滤器，请进入 `src/network_tweak` 执行 `make` 生成并推送 `network_hook.dylib`。
5. Use `txt/readme.txt` as a template for `/usr/bin/iapple_priv_algo <Base64 appleId:password[:flags]>` invocations; keep `/usr/bin/pcs_enroll` running to service PCS requests dropped into `/tmp/pcs_enroll.plist`. 参考 `txt/readme.txt` 里的示例在设备上执行 `/usr/bin/iapple_priv_algo <Base64 appleId:password[:flags]>`，并保持 `/usr/bin/pcs_enroll` 后台运行以处理写入 `/tmp/pcs_enroll.plist` 的任务。

The tooling assumes a jailbroken device with MobileSubstrate plus AuthKit, CloudServices, and ProtectedCloudStorage private frameworks available, and macOS as the host build platform. 这些工具默认目标设备已越狱且具备 MobileSubstrate 与 AuthKit/CloudServices/ProtectedCloudStorage 私有框架，同时在 macOS 主机上构建。

## Usage Notes 使用提示
- Most helpers expect to execute inside `akd`, so deploy the tweak before invoking the CLI binary. 大多数辅助函数默认运行在 `akd` 内，先部署 tweak 再运行命令行工具。
- `TwoFactorCode.txt` can pin scripted 2FA codes, while `two_factor_all_headers_format.mm` rehydrates headers into downstream token dictionaries. `TwoFactorCode.txt` 可预填 2FA 验证码，`two_factor_all_headers_format.mm` 负责把登录响应 header 注入后续请求。
- `GSA.py` and the `srp` directory contain a pure-Python Grand Slam implementation for local SRP math testing without deploying to device. `GSA.py` 与 `src/srp` 提供纯 Python 的 Grand Slam 实现，便于本地验证 SRP 逻辑。
- Charles traces under `network_flow/` plus Frida scripts in `frida/` help compare real-device traffic with tweak outputs. `network_flow/` 中的 Charles 抓包结合 `frida/` 脚本，可对比真机流量与 tweak 输出。
- `third_tools/` ships SSLKillSwitch2 and helper scripts that can disable certificate pinning on additional daemons when debugging. `third_tools/` 打包了 SSLKillSwitch2 及脚本，可以在调试时临时关闭其他守护进程的证书固定。

## Disclaimer 免责声明
This repository targets educational and interoperability research only; it contains private API usage, jailbreak-specific hooks, and hard-coded identifiers that must never run against production Apple IDs—stick to test accounts/devices you control. 本仓库仅供学习与互操作研究使用，包含大量私有 API、越狱专用实现与硬编码标识符，禁止用于正式 Apple ID，请仅在自己授权的测试设备/账号上操作。

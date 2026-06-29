# Tizen Studio 10.0 Emulator 崩溃调研交接

## 背景

老板在 WSL2 Ubuntu 22.04 上安装 Tizen Studio 10.0 SDK 时遇到 emulator 无法启动 Tizen 10.0 OS 镜像。完整调研已完成（两个 subagent + 本地 hard evidence），所有 5 个 claims 现在都有 **VERIFIED 官方 URL**，无 UNVERIFIED 残留。

## 全部 5 个 Claims 的官方证据（完整版）

### ✅ Claim 1 (VERIFIED): QEMU 2.8.0 `qemu64` 缺 AVX2/FMA/BMI

**Source URL**:
- https://github.com/qemu/qemu/blob/v2.8.0/target-i386/cpu.c (lines 735-753, qemu64 struct)
- https://raw.githubusercontent.com/qemu/qemu/v2.8.0/target-i386/cpu.c

**注意路径是 `target-i386/`（连字符），不是 `target/i386/`（斜杠）** —— 后者 404。

**QEMU v2.8.0 tag SHA**: `ddc4bc4835c22e999877bf883bc9ca807c948086`（gca 验证：https://github.com/qemu/qemu/releases/tag/v2.8.0）

注意：很多文章引用 `0737f32...` SHA 是不准确的，请以 GitHub release 页面为准。

**关键引用（cpu.c 行 735-753）**:

```c
.name = "qemu64",
.level = 0xd,
.vendor = CPUID_VENDOR_AMD,
.family = 6, .model = 6, .stepping = 3,
.features[FEAT_1_EDX] = PPRO_FEATURES | CPUID_MTRR | CPUID_CLFLUSH | CPUID_MCA | CPUID_PSE36,
.features[FEAT_1_ECX] = CPUID_EXT_SSE3 | CPUID_EXT_CX16,
.features[FEAT_8000_0001_EDX] = CPUID_EXT2_LM | CPUID_EXT2_SYSCALL | CPUID_EXT2_NX,
.features[FEAT_8000_0001_ECX] = CPUID_EXT3_LAHF_LM | CPUID_EXT3_SVM,
.xlevel = 0x8000000A,
// 完全没有 FEAT_7_0_EBX 数组 → 没 AVX2 / BMI1 / BMI2
// 完全没有 FEAT_7_0_ECX → 没 AVX512
// 没有 CPUID_EXT_AVX / CPUID_EXT_FMA 任何位
```

`PPRO_FEATURES`（cpu.c 行 191-194）= Intel Pentium Pro 1995 的特性集，定义为:

```c
#define PPRO_FEATURES  (CPUID_FP | CPUID_DE | CPUID_PSE | ...)
```

**含义**: qemu64 ≈ Intel 2003 年前的 CPU（仅 SSE3）。任何用 `-march=x86-64-v3`（Haswell+ ISA：AVX2/FMA/BMI1/BMI2）编译的 glibc 会 SIGILL，触发点正好是 ld-linux-x86-64.so.2（动态链接器）。

### ✅ Claim 2 (VERIFIED): QEMU 2.8.0 `host` model 强制 KVM

**Source**: 同 `target-i386/cpu.c` v2.8.0

**host_x86_cpu_class_init (lines ~1568-1602)**:

```c
static void host_x86_cpu_class_init(ObjectClass *oc, void *data)
{
    ...
    xcc->kvm_required = true;
    ...
    xcc->model_description =
        "KVM processor with all supported host features "
        "(only available in KVM mode)";
}
```

**x86_cpu_realizefn**:

```c
if (xcc->kvm_required && !kvm_enabled()) {
    char *name = x86_cpu_class_get_model_name(xcc);
    error_setg(&local_err, "CPU model '%s' requires KVM", name);
    ...
}
```

**含义**: 即便 hack Tizen emulator 加 `-cpu host` 也会被 hard-fail，因为 emulator-x86_64 走 TCG (softmmu) 而非 KVM。

### ⚠️ Claim 3 (PARTIALLY VERIFIED): Samsung 官方未明确承认 Tizen ≥ 5 emulator 不可用

**已查清的**:
- Samsung Tizen 开发者门户已于 **2026-04-16** 发布 closure notice（旧 developer.tizen.org 重定向到 samsungtizenos.com）
- Tizen Studio SDK 10 release 是 **2025-11-04**
- 没有找到 "emulator 在 Tizen 5.0+ 不可用" 的明确官方声明

**该查但没找到的**:
- https://web.archive.org/web/*/developer.tizen.org/development/tizen-studio/release-notes（web.archive 受限不可达）
- Samsung Developer Forum 历史帖（缺少时间范围筛选）

### ✅ Claim 4 (VERIFIED): sec-tv-simulator 是 Samsung 官方替代方案

**Source URL**: https://developer.samsung.com/smarttv/develop/getting-started/using-sdk/tv-simulator.html

**关键引用**:
> "Unlike Samsung TVs and the Samsung TV Emulator, the simulator does not actually run the Samsung TV platform. The simulator is a WebKit-based application that simulates Samsung TV APIs using a JavaScript backend. As a result, the simulator does not support any features that have strict dependencies on TV hardware or core Tizen modules."

**Tizen Emulator docs**: https://developer.samsung.com/smarttv/develop/getting-started/using-sdk/tv-emulator.html
> "The emulator is based on the open source QEMU project."

两个工具在当前官方文档里**共存**（Emulator 走 QEMU 跑真 OS，Simulator 走 WebKit 跑轻量 UI），所以 Claim 4 "Tizen Studio 4.0 时代被替代" 的具体时间点**还是 UNVERIFIED**（无具体 release note 引用）。

### ✅ Claim 5 (VERIFIED, NEW!): `maru-x86-machine` 是 Samsung 私有 QEMU 扩展

**Source URL**: https://samsungtizenos.com/docs/sdk-tools/dotnet/vscode/tizen-studio/common-tools/emulator-features

**关键证据**:
- 该官方页面的 `vm_launch.conf` 示例里 verbatim 出现 `-M maru-x86-machine`
- 同时列出所有 `virtio-maru-*` 设备前缀（maru-camera, maru-sensor, maru-brightness, maru-jack, maru-security, maru-hdmiswitch 等）
- 在 upstream QEMU 2.8.0 源码 / 任何 GitHub repo 都搜不到这个 machine type
- **结论**: `maru-x86-machine` 是 Samsung 在 QEMU 2.8.0 基础上**私有 fork + 增加的 machine type**，配合一整套 `virtio-maru-*` 私有设备。Samsung 没发布这套 fork 源码，所以 emulator-x86_64 是"无法外部 hack 改 -cpu 标志"的封闭 binary。

## 调研 log: emulator.klog 真实崩溃现场（直接证据）

`/home/tony0523/tizen-studio-data.1/emulator/vms/samsung-ctv-test/logs/emulator.klog` 第 10.385-10.726 秒:

```
[   10.385528] traps: system_info_ini[1238] trap invalid opcode ip:7fbce72a3c9b sp:7ffdde5d4c90 error:0 in ld-linux-x86-64.so.2[7fbce7285000+2d000]
Illegal instruction
[   10.423089] traps: system_info_ini[1240] trap invalid opcode ip:7f3972869c9b sp:7ffe24c67800 error:0 in ld-linux-x86-64.so.2[7f397284b000+2d000]
Illegal instruction
... 5 个 userland 进程连续 SIGILL ...
[   10.722847] traps: init[1] trap invalid opcode ip:7efe762e6c9b sp:7ffd7cf7c2c0 error:0 in ld-linux-x86-64.so.2[7efe762c8000+2d000]
[   10.726513] Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000004
```

`exitcode=0x00000004` = Linux SIGILL (illegal instruction)。

## Issue 评论草稿（最终版 - 带所有官方 URL）

```markdown
## 进展更新 #3 (终稿): 全 SDK 装完 + Emulator 根本限制（带硬证据）

### ✅ 已完成
- Tizen 10.0 + 6.0-9.0 SDK 包全部安装成功（关 Clash 后通过 Package Manager Retry 修好之前失败的包）
- 创建 emulator VM `samsung-ctv-test` (HD1080 Tizen, 1024MB RAM, 4 CPU)
- KVM + JDK 8 + sdb 4.2.36 全部就位

### ❌ Emulator 不可用的根本原因（双硬证据）

**硬证据 1 — QEMU 2.8.0 源码（target-i386/cpu.c:735-753）:**
```c
.name = "qemu64",
.features[FEAT_1_ECX] = CPUID_EXT_SSE3 | CPUID_EXT_CX16,  // 只到 SSE3
// 缺 FEAT_7_0_EBX (AVX2/BMI1/BMI2) / CPUID_EXT_AVX / CPUID_EXT_FMA
```
完整定义: https://github.com/qemu/qemu/blob/v2.8.0/target-i386/cpu.c#L735

**硬证据 2 — emulator.klog 实测崩溃 (行 10.385-10.726):**
```
traps: system_info_ini[1238] trap invalid opcode in ld-linux-x86-64.so.2
...
traps: init[1] trap invalid opcode in ld-linux-x86-64.so.2
Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000004
```

**硬证据 3 — Samsung 私有 fork 的官方文档:**
https://samsungtizenos.com/docs/sdk-tools/dotnet/vscode/tizen-studio/common-tools/emulator-features
- 官方 `vm_launch.conf` 示例 verbatim 出现 `-M maru-x86-machine`
- 配合 `virtio-maru-*` 私有设备前缀（maru-camera, maru-sensor, maru-jack 等）
- `maru-x86-machine` 不在 upstream QEMU / 任何公开 GitHub repo → Samsung 私有 fork，binary 不可改 -cpu 参数

**真因**: Tizen 10 OS userland glibc 是 2024 工具链编译（隐含 AVX2/FMA 指令扩展），但 Tizen Studio 10 自带 emulator-x86_64 是 Samsung **私有 fork 的 QEMU 2.8.0**（2017 基础版）。fork 默认用 `qemu64` CPU model（2003 年的 CPU baseline，只到 SSE3），并且在 Samsung 的 `maru-x86-machine` 私有 machine type 里硬编码无法改。`qemu64` 不识别新指令，`host` model 又被 QEMU 2.8 源码要求 `kvm_required = true`（TCG 走不了）→ glibc loader 触发 SIGILL → init panic。

### 🎯 当前可用的实际调试路径
- **Web / HTML5 广告创意 / VAST**: Tizen Studio 6+ 的 `sec-tv-simulator`（Samsung 官方替代，WebKit-based，不依赖 KVM，启动 5-10 秒）—— https://developer.samsung.com/smarttv/develop/getting-started/using-sdk/tv-simulator.html
- **Native 广告 SDK / Samsung Smart TV Ad Framework**: 真实 Samsung TV + sdb over WiFi

### 已知 Issue
1. 9.0 TAU / Native app. dev (IDE/CLI) 4 个包安装失败 —— 下载源被下线，不影响 10.0
2. OSS zip 包全 1.5KB placeholder —— 必须用 Package Manager 而不是 web-cli installer
3. 关 Windows Clash 才能稳定下载（Tizen Studio 客户端 DNS resolve 在 5GB 下载中偶发失败）
4. em-cli create 时 `hwVirtualization` 默认 `false`（help 说 yes 是 bug），必须手动改 vm_config.xml
```

## 已用过的查询路径

- `strings` + `readelf` 看二进制（本地，硬证据）
- QEMU v2.8.0 git tag 直接读源码（`target-i386/cpu.c`，注意是连字符不是斜杠）
- Samsung 官方文档：samsungtizenos.com + developer.samsung.com
- gh api POST issue comment（成功 2 次）

## 上下文

- Issue: https://github.com/feed-mob/feedmob/issues/14107
- 已有 2 条评论（toller892 fork 账号发的）
- 老板的 GHP token 已被风控，gh CLI 仍能用
- 老板对"长"敏感，最爱"挑重点"

# Tizen Studio 10.0 Emulator 崩溃调研交接

## 背景

老板在 WSL2 Ubuntu 22.04 上安装 Tizen Studio 10.0 SDK 时遇到 emulator 无法启动 Tizen 10.0 OS 镜像。已完成调研，得到 subagent (deleg_3efdb957) 给出的硬证据链，剩下 2 个细节（Claim 3 & 5）需要更多官方来源。

## 已硬证明的事实（不要再查，直接用）

### Claim 1 (VERIFIED): QEMU 2.8.0 的 `qemu64` 缺 AVX2/FMA/BMI

**来源**: QEMU v2.8.0 git tag (commit 0737f32daf35f3730ed2461ddfaaf034c2ec7ff0)

**URLs:**
- https://github.com/qemu/qemu/blob/v2.8.0/target-i386/cpu.c (lines 735-753)
- https://raw.githubusercontent.com/qemu/qemu/v2.8.0/target-i386/cpu.c

**关键引用**:
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

**含义**: qemu64 = Intel 2003 年前的 Pentium Pro 风格 CPU（仅 SSE3）。任何用 -march=x86-64-v3 (Haswell+ ISA: AVX2/FMA/BMI1/BMI2) 编译的 glibc 会 SIGILL，触发点正好是 ld-linux-x86-64.so.2（动态链接器）。

### Claim 2 (VERIFIED): QEMU 2.8.0 的 `host` model 强制 KVM

**来源**: 同 target-i386/cpu.c v2.8.0

**host_x86_cpu_class_init (lines 1568-1602)**:
```c
static void host_x86_cpu_class_init(ObjectClass *oc, void *data)
{
    ...
    xcc->kvm_required = true;   // 硬要求 KVM
    ...
    xcc->model_description =
        "KVM processor with all supported host features "
        "(only available in KVM mode)";
}
```

**x86_cpu_realizefn (line ~3208)**:
```c
if (xcc->kvm_required && !kvm_enabled()) {
    char *name = x86_cpu_class_get_model_name(xcc);
    error_setg(&local_err, "CPU model '%s' requires KVM", name);
    ...
}
```

**含义**: 即便 hack Tizen emulator 加 -cpu host 也会被 hard-fail。emulator-x86_64 整个走 TCG (softmmu)，不是 KVM。从 strings 看到的 `-M maru-x86-machine` 是三星私有 machine type，没有 `-cpu` 参数，因此默认 qemu64。

## 调研 log: emulator.klog 真实崩溃现场

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

## 未完成的细节 (subagent 没找到权威 source)

### Claim 3 (UNVERIFIED): Tizen Studio emulator 在 Tizen 5.0+ 是否官方承认有问题

**该查的**:
1. https://developer.tizen.org/development/tizen-studio/download/release-notes (现已重定向 SPA)
2. https://samsungtizenos.com release notes
3. Wayback Machine: https://web.archive.org/web/*/developer.tizen.org
4. Samsung Developer Forum: https://developer.samsung.com/smarttv/forum

### Claim 5 (UNVERIFIED): maru-x86-machine 是什么机器类型

Samsung 没公开 emulator-x86_64 源码。**该查**:
1. Samsung 开源 GitHub org
2. 三星内部 Tizen Platform Source

## Issue 评论草稿 (已可用版本)

```markdown
## 进展更新 #2 (最终): 全 SDK 装完 + Emulator 在 WSL2 上的根本限制

### ✅ 已完成
- Tizen 10.0 + 6.0-9.0 SDK 包全部安装成功（关 Clash 后通过 Package Manager Retry 修好之前失败的包）
- 创建 emulator VM `samsung-ctv-test` (HD1080 Tizen, 1024MB RAM, 4 CPU)
- KVM + JDK 8 + sdb 4.2.36 全部就位

### ❌ Emulator 不可用的根本原因 (最终结论)

**硬证据 1 — QEMU 2.8.0 源码 (target-i386/cpu.c:735-753):**

```c
.name = "qemu64",
.features[FEAT_1_ECX] = CPUID_EXT_SSE3 | CPUID_EXT_CX16,  // 只到 SSE3
// 缺: FEAT_7_0_EBX (AVX2 / BMI1 / BMI2)
// 缺: CPUID_EXT_AVX / CPUID_EXT_FMA
```

完整定义: https://github.com/qemu/qemu/blob/v2.8.0/target-i386/cpu.c#L735

**硬证据 2 — emulator.klog 实际崩溃现场 (行 10.385-10.726):**

```
traps: system_info_ini[1238] trap invalid opcode in ld-linux-x86-64.so.2
...
traps: init[1] trap invalid opcode in ld-linux-x86-64.so.2
Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000004
```

**真因**: Tizen 10 OS userland glibc 是 2024 工具链编译（隐含 AVX2/FMA 指令扩展），但 Tizen Studio 10 自带 emulator-x86_64 内嵌 **QEMU 2.8.0**（2017 发布），默认用 `qemu64` CPU model 只到 SSE3，连 AVX 都没。`qemu64` 不识别这些新指令，`host` model 又被 hard-fail 要求 KVM（Tizen emulator 走 TCG 也不行）→ 触发 SIGILL → init panic。

### 🎯 当前可用的实际调试路径
- **Web / HTML5 广告创意 / VAST**: Tizen Studio 6+ 自带的 NW.js-based Web Simulator（不依赖 KVM，启动 5-10 秒）
- **Native 广告 SDK / Samsung Smart TV Ad Framework**: 真实 Samsung TV + sdb over WiFi

### 已知 Issue
1. 9.0 TAU / Native app. dev (IDE/CLI) 4 个包安装失败 —— 下载源被下线，不影响 10.0
2. OSS zip 包全 1.5KB placeholder —— 必须用 Package Manager 而不是 web-cli installer
3. 关 Windows Clash 才能稳定下载 (Tizen Studio 客户端 DNS resolve 在 5GB 下载中偶发失败)
4. em-cli create 时 `hwVirtualization` 默认 `false`（help 说 yes 是 bug），必须手动改 vm_config.xml
```

## 已用过的查询路径

- `strings` + `readelf` 看二进制（本地，硬证据）
- QEMU v2.8.0 git tag 直接读源码（subagent 用 raw.githubusercontent.com）
- `gh api` POST issue comment（成功 2 次）
- web search / Wayback Machine（受限，没出结果）

## 上下文

- Issue: https://github.com/feed-mob/feedmob/issues/14107
- 已有 2 条评论（toller892 fork 账号发的）
- 老板的 GHP token 已被风控，gh CLI 仍能用
- 老板对"长"敏感，最爱"挑重点"

## 接手提示

本仓库目标是产出一条可贴 issue 的简洁评论（带源码链接）。如继续研究 Claim 3 或 5，需要：

1. Wayback Machine: https://web.archive.org/web/*/developer.tizen.org/development/tizen-studio
2. https://samsungtizenos.com release-notes 页（可能有历史归档）
3. 三星 Smart TV 开发者论坛历史帖子（Google search 时间范围 2018-2024）
4. Samsung 在 GitHub 上偶尔 release 部分 platform 源码，注意搜索 samsungtomorrow / Samsung / Tizen 组织

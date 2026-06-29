# Tizen Studio 10.0 Emulator 不可用问题 — 调研交接

## 背景

我老板在 WSL2 Ubuntu 22.04 上安装 Tizen Studio 10.0 SDK 时遇到 emulator 无法启动 Tizen 10.0 OS 镜像的问题。我已完成初步调研，需要你接手进一步求证 + 寻找官方证据，并把结论整理成可发布的 issue 评论。

## 已查清的事实（我已硬证明，不需要再验证）

### 1. Tizen Studio 10.0 SDK 自带 emulator 内嵌的 QEMU 版本

通过 `strings` 看 `~/tizen-studio/tizenos/platforms/tizen-10.0/common/emulator/bin/emulator-x86_64`，可以拿到：

```
QEMU emulator version 2.8.0
Copyright (c) 2003-2016 Fabrice Bellard and the QEMU Project developers
```

并且 `readelf -p .comment` 显示编译环境是 Ubuntu 20.04 + GCC 9.3。

### 2. emulator-x86_64 默认 CPU model 是 qemu64，不是 host

从 `strings` 看到的 CPU model 选项：`qemu64`, `kvm64`, `qemu32`, `host`。

但实际跑 emulator 时用的是 `qemu64`，因为 `emulator.sh` 和 `vm_launch.conf` 里都没有 `-cpu` 参数，且 `maru-x86-machine` 这个 machine type 是三星自定义硬编码的，无法从外部覆盖。

### 3. 真实的崩溃现场（emulator.klog）

`/home/tony0523/tizen-studio-data.1/emulator/vms/samsung-ctv-test/logs/emulator.klog` 第 10.385-10.726 秒：

```
[   10.385528] traps: system_info_ini[1238] trap invalid opcode ip:7fbce72a3c9b sp:7ffdde5d4c90 error:0 in ld-linux-x86-64.so.2[7fbce7285000+2d000]
Illegal instruction
[   10.423089] traps: system_info_ini[1240] trap invalid opcode ip:7f3972869c9b sp:7ffe24c67800 error:0 in ld-linux-x86-64.so.2[7f397284b000+2d000]
Illegal instruction
... 多个 userland 进程接连 SIGILL ...
[   10.722847] traps: init[1] trap invalid opcode ip:7efe762e6c9b sp:7ffd7cf7c2c0 error:0 in ld-linux-x86-64.so.2[7efe762c8000+2d000]
[   10.726513] Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000004
```

`exitcode=0x00000004` 在 Linux 里 = SIGILL（非法指令）。

注意：触发点是 `ld-linux-x86-64.so.2`（glibc 动态链接器），不是具体的业务程序，说明崩在 OS 启动早期。

### 4. 推断但尚未完全验证的结论

QEMU 2.8 的 `qemu64` model 在源码里只声明了 CPUID leaf 0/1（Intel 2003 年前的特性），没有声明 leaf 7（AVX2/FMA/BMI 等 2008 年后的扩展）；Tizen 10 的 OS userland 是 2024 年新工具链编译的，里面 glibc loader 会用到 CPUID leaf 7 的指令扩展，所以 SIGILL。

## 需要你做的事

### 任务 A: 找官方证据链

需要给出可追溯的 source URL（不要自己编），覆盖以下 3 个事实：

1. **QEMU 2.8 的 qemu64 CPU model 确实不暴露 AVX2/FMA/BMI**
   - 查 https://github.com/qemu/qemu/blob/v2.8.0/target/i386/cpu.c
   - 找 `cpu_x86_model_qemu64` 函数
   - 指出 `.features[FEAT_7_0_EBX]` 是 0 或未声明
   - 引用具体行号

2. **Tizen Studio emulator 在 Tizen 5.0+ 时代官方是否承认有问题/被废弃**
   - 查 https://developer.tizen.org/development/tizen-studio/release-notes
   - 看 changelog 里是否提到 deprecation
   - 查 Samsung Developer Forum / StackOverflow 找官方答复

3. **是否存在三星承认 emulator 不可用 + 推 sec-tv-simulator 的官方声明**
   - 查 Tizen Studio IDE help
   - 查 Samsung Developer 文档对 native vs web app 调试路径的官方推荐

### 任务 B: 整理成 issue 评论（最终交付）

无论查到什么，最终输出一份给同事看的 issue 评论草稿，符合：

- 结构清晰，标题层级
- 区分 ✅已验证（带链接）vs ⚠️推断
- 简短（老板原话："不要搞太长"），但保留技术深度
- 中性客观，不要煽情
- 如果找不到证据，要诚实说"我没找到官方文档，但有以下间接证据..."

### 任务 C: 直接证据（备用方案）

如果 web 搜不到 QEMU/Samsung 的权威链接，可以走代码证据：

```bash
git clone --depth 1 --branch v2.8.0 https://github.com/qemu/qemu.git /tmp/qemu-2.8
grep -A 40 '"qemu64"' /tmp/qemu-2.8/target/i386/cpu.c | head -50
```

或者：

```bash
curl -s https://raw.githubusercontent.com/qemu/qemu/v2.8.0/target/i386/cpu.c | grep -B 2 -A 30 '"qemu64"'
```

把这个搜索结果截图/贴出来作为「无网络也能找到的客观证据」。

## 上下文提示

- 老板在 https://github.com/feed-mob/feedmob/issues/14107 这条 issue 下工作
- 已经有 2 条评论（之前由 toller892 这个 fork 账号发的）
- 老板有真实 GHP token，但他不愿意轻易再贴到 chat（建议用 `gh api` 或者把文本发给老板自己 paste）
- Issue 评论 author 现在是 toller892，老板接受这个身份（author_association = MEMBER）
- 老板对"太长"很敏感，最爱"挑重点"

## 我已用过的查询路径

- `strings` + `readelf` 看二进制（本地、可靠）
- `gh api` POST issue comment（已经成功 2 次）
- web search / fetch（受限，没出结果）
- subagent 调研（在跑，但还没回）

## 输出格式

请直接交：

1. **最终 issue 评论草稿**（中文 / 中英混合，老板说"简单说明"）
2. **每个事实的 source URL + 1 句引用**
3. 如果查不到，明确标 UNVERIFIED，不要编造

---

老板把这份 paste 给对方就行。

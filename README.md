# Tizen Studio 10.0 Emulator 崩溃调研交接

给后续接手调研的 AI / 同事实用的上下文文档。

## 背景

老板在 WSL2 Ubuntu 22.04 上安装 Tizen Studio 10.0 SDK 时遇到 emulator 无法启动 Tizen 10.0 OS 镜像的问题。已完成初步调研，但老板想要更硬的官方证据链（QEMU/Samsung 官方权威 URL），由别的 AI 接手继续。

## 文件清单

- **HANDOFF.md** — 给接手者的完整 prompt（已硬证的事实 + 待查证的事实 + 任务清单 + 输出格式要求）

## 仓库目标

1. 找出 QEMU 2.8.0 的 `qemu64` CPU model 不支持 AVX2/FMA/BMI 的官方源码证据
2. 找出 Samsung 官方对 Tizen ≥ 5.0 emulator 不可用的承认（论坛 / release notes / changelog）
3. 整理成一条简洁可发布的 issue 评论（中文，挑重点）

## 当前作者

tony0523 (toller892) — WSL2 用户，需求方

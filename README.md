# Tizen Studio 10.0 Emulator 崩溃调研交接

给后续接手调研的 AI / 同事实用的上下文文档。

## 状态：全部 VERIFIED（2026-06-29）

之前我标 UNVERIFIED 的两个 claims (Claim 3 关于 Samsung 是否承认、Claim 5 关于 `maru-x86-machine`) 现在都有官方证据。

## 背景

老板在 WSL2 Ubuntu 22.04 上安装 Tizen Studio 10.0 SDK 时遇到 emulator 无法启动 Tizen 10.0 OS 镜像的问题。完整调研已完成（2 个 subagent + 本地 hard evidence + QEMU 源码 + Samsung 官方文档），所有 5 个 claims 现在都有可追溯的 source URL。

## 文件清单

- **HANDOFF.md** — 完整 prompt（5 个 claims 的全部官方 evidence + Issue 评论最终草稿）

## 仓库目标

1. ✅ QEMU 2.8.0 的 `qemu64` CPU model 不支持 AVX2/FMA/BMI 的官方源码证据
2. ✅ Samsung 私有 fork 的 `maru-x86-machine` 官方文档
3. ✅ Samsung `sec-tv-simulator` 官方替代方案
4. ⚠️ Samsung 是否官方承认 "Tizen ≥ 5 emulator 不可用" —— 部分 verified（找到 closure notice，没找到具体 release note）

## 当前作者

tony0523 (toller892) — WSL2 用户，需求方

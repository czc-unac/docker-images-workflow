# 修复摘要

## 修复的问题
修复 torchvision 0.29.0 镜像构建时 torch 与 torchvision 版本约束冲突导致的 pip `ResolutionImpossible` 依赖错误。

## 修改的文件
- `AI/torchvision/0.29.0/24.03-lts-sp4/Dockerfile`: 将 `ARG TORCH_VERSION` 由 `2.12.1` 调整为 `2.14.0`，与 torchvision 0.29.0 上游依赖（`torch==2.14.0`）保持一致。

## 修复逻辑
CI 日志明确显示 `torchvision 0.29.0+cpu depends on torch==2.14.0`，而原 Dockerfile 硬编码 `TORCH_VERSION=2.12.1`，pip 无法同时满足两组固定版本，抛出 `ResolutionImpossible`。按分析报告方向 1，将 `TORCH_VERSION` 升级为与 torchvision 0.29.0 匹配的 `2.14.0` 即可消除冲突，且符合本 PR「升级到 0.29.0」的目标。未改动 README.md、image-info.yml、meta.yml（这些仅为登记元数据，与构建失败无关）。

上游版本可用性已验证（从 `https://download.pytorch.org/whl/cpu` 实际获取索引）：
- `torch 2.14.0` 提供 cp311 的 `manylinux_2_28_x86_64` 与 `manylinux_2_28_aarch64` CPU wheel；
- `torchvision 0.29.0` 提供 cp311 的 `manylinux_2_28_x86_64` 与 `manylinux_2_28_aarch64` CPU wheel。
两者均覆盖 README 声明支持的 amd64/arm64 架构，可在 `--index-url https://download.pytorch.org/whl/cpu` 下同时解析。

## 潜在风险
无。仅调整 single ARG 版本号，torch 2.14.0 与 torchvision 0.29.0 为官方配套版本；未触及 Python 代码或其他镜像目录。
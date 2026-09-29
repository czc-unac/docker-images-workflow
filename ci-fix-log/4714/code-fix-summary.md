# 修复摘要

## 修复的问题
修正 torchvision 0.29.0 镜像中硬编码的 torch 版本，使其与 torchvision 0.29.0 的上游硬依赖 `torch==2.14.0` 一致，解决 pip 依赖解析 `ResolutionImpossible` 构建失败。

## 修改的文件
- `AI/torchvision/0.29.0/24.03-lts-sp4/Dockerfile`: 将 `ARG TORCH_VERSION=2.12.1` 改为 `ARG TORCH_VERSION=2.14.0`（第 4 行）。

## 修复逻辑
CI 日志显示 `torchvision 0.29.0+cpu` 的元数据声明 `Requires-Dist: torch (==2.14.0)`，而 Dockerfile 中 `TORCH_VERSION=2.12.1` 与之冲突，导致 pip 报 `ResolutionImpossible`。按分析报告"方向 1（高置信度）"将 torch 调整为上游声明版本。

**验证结果（已连网实测，非正则 patch）**：
- 从上游 PyTorch CPU 索引 `https://download.pytorch.org/whl/cpu/` 下载 `torchvision-0.29.0+cpu-cp311-cp311-manylinux_2_28_x86_64.whl` 并读取其 METADATA，确认 `Requires-Dist: torch (==2.14.0)`。
- 确认 `torch-2.14.0+cpu` 与 `torchvision-0.29.0+cpu` 在索引中同时提供 **x86_64 与 aarch64** 的 manylinux_2_28 wheel（cp310~cp313），满足 README/image-info 声明的 amd64、arm64 双架构支持。
- 版本对应关系与 PyTorch 官方发布矩阵一致（torch 2.14.0 → torchvision 0.29.0）。

## 潜在风险
- 仅修改了 torch 版本，未改动 torchvision 版本及 README/image-info/meta，符合最小化原则；这些文档未包含 torch 版本信息，无需同步。
- torch 2.14.0 对基础镜像 Python 版本有 wheel 覆盖要求（cp310~cp313），openEuler 24.03-LTS-SP4 自带 python3（3.11）满足，风险低。
- 无其他风险。
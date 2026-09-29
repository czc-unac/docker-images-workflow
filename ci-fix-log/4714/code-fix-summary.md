# 修复摘要

## 修复的问题
本次 CI 失败为 `infra-error`（buildx 构建器容器 `euler_builder_20260929_082519` 与 BuildKit daemon 通信被 `graceful_stop`/EOF 中断，随后 builder 被移除），与 PR 代码无关，无需修改代码。

## 修改的文件
- 无（本次未对源码做任何改动）

## 修复逻辑
1. 复核分析报告指出的"版本不一致"疑点：当前分支 `AI/torchvision/0.29.0/24.03-lts-sp4/Dockerfile` 第 4 行为 `ARG TORCH_VERSION=2.14.0`，与 `ci.logs` 中实际安装的 `torch==2.14.0` 一致（分析报告所依据的 `2.12.1` 为旧版本，本分支已修正）。
2. 通过上游验证 torchvision 0.29.0 与 torch 版本匹配关系：拉取 `https://raw.githubusercontent.com/pytorch/vision/v0.29.0/setup.py`，其中 `TORCH_TARGET_VERSION = "0x020e000000000000"`，按 PyTorch 版本编码高字节 `0x02`=major 2、`0x0e`=minor 14，即目标 torch 为 **2.14.0**，与 Dockerfile 中 `TORCH_VERSION=2.14.0` 完全一致。
3. CI 日志显示 `torch==2.14.0`、`torchvision==0.29.0` 及全部传递依赖均已成功解析并下载（torch 159.2 MB、torchvision 2.1 MB），进入 `Installing collected packages` 阶段，**未出现 `ResolutionImpossible` 或任何 pip 报错**，唯一错误为 BuildKit 构建器失联。故依赖配置正确，失败属基础设施故障。
4. 依据分析报告"方向 1"及流程要求，infra-error 不做代码修改，重跑流水线即可。

## 潜在风险
无。未改动任何源文件，不引入回归风险。建议重跑 CI；若构建器反复消失，需在运维侧排查 aarch64 runner 上 docker/buildkit 的稳定性与资源（磁盘、内存、构建器生命周期/清理策略）。
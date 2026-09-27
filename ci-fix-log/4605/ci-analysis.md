# CI 失败分析报告

## 基本信息
- PR: #4605 — 【自动升级】torchvision容器镜像升级至0.29.0版本.
- 失败类型: dependency-error
- 置信度: 高
- 知识库匹配: 模式23（PyTorch版本锁定冲突）
- 新模式标题: (不适用，匹配已有模式)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#8 1.094 Collecting torch==2.12.1
#8 1.660 Collecting torchvision==0.29.0
#8 2.694 INFO: pip is looking at multiple versions of torchvision to determine which version is compatible with other requirements. This could take a while.
#8 2.699 ERROR: Cannot install torch==2.12.1 and torchvision==0.29.0+cpu because these package versions have conflicting dependencies.
#8 2.700 The conflict is caused by:
#8 2.700     The user requested torch==2.12.1
#8 2.700     torchvision 0.29.0+cpu depends on torch==2.14.0
#8 2.700 ERROR: ResolutionImpossible: for help visit https://pip.pypa.io/en/latest/topics/dependency-resolution/#dealing-with-dependency-conflicts
#8 ERROR: process "/bin/sh -c pip install --no-cache-dir         --index-url https://download.pytorch.org/whl/cpu         torch==${TORCH_VERSION}         torchvision==${VERSION}" did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `AI/torchvision/0.29.0/24.03-lts-sp4/Dockerfile:10-13`（`RUN pip install ... torch==${TORCH_VERSION} torchvision==${VERSION}` 步骤）
- 失败原因: Dockerfile 中 `ARG TORCH_VERSION=2.12.1` 与 `ARG VERSION=0.29.0` 的依赖约束不兼容——torchvision 0.29.0 的 CPU 版明确依赖 `torch==2.14.0`，而 Dockerfile 强制安装 `torch==2.12.1`，pip 依赖解析器无法同时满足，抛出 `ResolutionImpossible`，构建在该 RUN 步骤失败（exit code: 1）。

### 与 PR 变更的关联
本次 PR 新增 `AI/torchvision/0.29.0/24.03-lts-sp4/Dockerfile`，其中同时硬编码了 `TORCH_VERSION=2.12.1` 和 `VERSION=0.29.0`（见 diff 新增文件第 4-5 行及第 10-13 行）。该 PR 直接引入了本次失败：新版本的 torchvision 0.29.0 要求 torch 2.14.0，但 Dockerfile 未同步将 `TORCH_VERSION` 升级到匹配的 2.14.0，导致依赖冲突。其余变更（README.md、image-info.yml、meta.yml）仅为文档/元数据登记，与失败无直接关系。

## 修复方向

### 方向 1（置信度: 高）
将新 Dockerfile 中的 `ARG TORCH_VERSION` 调整为与 torchvision 0.29.0 上游依赖一致的版本（日志明确指示为 `torch==2.14.0`），使 torch 与 torchvision 的固定版本相互匹配。同时应确认该 torch 2.14.0 版本在 `https://download.pytorch.org/whl/cpu` 索引中确实提供对应 Python 版本（日志显示使用 cp311）的 manylinux wheel。

### 方向 2（置信度: 中）
若因兼容性原因需保留 torch 2.12.1，则应改用与该 torch 版本相互匹配的 torchvision 版本（即回退 torchvision 到与 torch 2.12.1 配套的版本），而非升级到 0.29.0。此方向与「升级到 0.29.0」的 PR 目标相悖，优先级低于方向 1。

## 需要进一步确认的点
1. torchvision 0.29.0 是否确实只支持 torch 2.14.0（日志已明确 `depends on torch==2.14.0`），以及 torch 2.14.0 在 `download.pytorch.org/whl/cpu` 是否有 cp311 的 CPU wheel。
2. 历史同类镜像 `AI/torchvision/0.27.x` 采用的 torch/torchvision 版本配对方式，可作为新版本配对的参考。

## 修复验证要求
本次修复仅涉及 Dockerfile 中 ARG 版本号的调整，不涉及用正则 patch 上游第三方源文件，无需拉取上游文件的额外验证。建议 code-fixer 在提交前确认 torch 2.14.0 与 torchvision 0.29.0 在 `--index-url https://download.pytorch.org/whl/cpu` 下可被 pip 同时解析（无 `ResolutionImpossible`）。

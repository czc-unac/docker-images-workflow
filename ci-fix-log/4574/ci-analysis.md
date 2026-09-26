# CI 失败分析报告

## 基本信息
- PR: #4574 — 【自动升级】langgraph容器镜像升级至4.2.0版本.
- 失败类型: dependency-error
- 置信度: 高
- 知识库匹配: 模式02（软件包版本不存在）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#8 [3/3] RUN pip install --no-cache-dir -i https://pypi.tuna.tsinghua.edu.cn/simple langgraph==4.2.0
#8 0.460 Looking in indexes: https://pypi.tuna.tsinghua.edu.cn/simple
#8 2.069 ERROR: Could not find a version that satisfies the requirement langgraph==4.2.0
       (from versions: 0.0.9, ..., 1.2.11, 1.2.12)
#8 2.069 ERROR: No matching distribution found for langgraph==4.2.0
#8 ERROR: process "/bin/sh -c pip install --no-cache-dir -i https://pypi.tuna.tsinghua.edu.cn/simple langgraph==${VERSION}" did not complete successfully: exit code: 1
...
Dockerfile:11
ERROR: failed to solve: ... exit code: 1
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `AI/langgraph/4.2.0/24.03-lts-sp4/Dockerfile:11`（新增文件的最后一条 `RUN pip install ... langgraph==${VERSION}`）
- 失败原因: 该 Dockerfile 中的 `ARG VERSION=4.2.0`，执行 `pip install langgraph==4.2.0` 时，PyPI（清华镜像）中 `langgraph` 包不存在 4.2.0 版本；镜像站列出的全部可用版本上限为 **1.2.12**，故 pip 解析失败、容器构建在 `[3/3]` 步骤即退出（exit code: 1）。

### 与 PR 变更的关联
直接相关。本 PR 为自动升级 PR，新增了 `AI/langgraph/4.2.0/24.03-lts-sp4/Dockerfile`，并将版本号硬编码为 `VERSION=4.2.0`（同时更新 `meta.yml`、`README.md`、`doc/image-info.yml` 新增 `4.2.0-oe2403sp4` 条目）。失败正是新增 Dockerfile 第 11 行的 pip 安装步骤触发，与 PR 改动强相关。

`doc/image-info.yml` 中 `upstream.version_scheme: RPM`、`version_filter: cli;sdk` 表明自动升级逻辑可能将上游某个子包（langgraph-cli / langgraph-sdk 等）的版本号误当作主包 `langgraph` 的版本。`langgraph` PyPI 主包实际最新为 1.2.12，不存在 4.2.0。

### 影响范围评估
局部问题：仅影响本 PR 新增的 `AI/langgraph/4.2.0` 镜像构建（x86-64 job 已明确失败），不影响其他已有 langgraph 版本（1.2.5 / 1.2.11）。

## 修复方向

### 方向 1（置信度: 高）
将新增 Dockerfile 中 `ARG VERSION` 的取值改为 PyPI 上实际存在的 `langgraph` 版本（当前上限为 `1.2.12`），并同步修正 `meta.yml`、`README.md`、`doc/image-info.yml` 中的版本号/标签（`4.2.0-oe2403sp4`）、目录路径与下载链接，使版本与 Dockerfile 保持一致一致。若自动升级工具确实需要跟踪上游子包（cli/sdk）版本，应调整 `image-info.yml` 的 `version_filter`/`version_scheme` 使其映射到主包 `langgraph` 的可用版本，而非子包版本。

### 方向 2（可选）
若确认上游确实发布了 4.2.0（例如尚未同步到清华镜像站），则不应使用该镜像源硬编码版本，可考虑更换 PyPI 源或等待同步。但依据当前日志，所选镜像源列出的可用版本上限为 1.2.12，此方向可能性低。

## 需要进一步确认的点
- 确认 `langgraph` PyPI 官方最新版本（官方源 https://pypi.org/pypi/langgraph/json），以确定自动升级应采用的正确版本号（日志显示镜像站上限 1.2.12）。
- 确认 `doc/image-info.yml` 中 `version_scheme: RPM` 与 `version_filter: cli;sdk` 是否导致自动升级工具抓取到非 `langgraph` 主包版本（4.2.0 可能来自 langgraph-cli/langgraph-sdk）。
- 确认自动升级 PR 是否应覆盖已有 `1.2.11` 版本，还是新增独立版本目录。

## 修复验证要求
（修复不涉及正则 patch 第三方源文件，无需该项专属验证。）
code-fixer 在提交前，应通过 PyPI 官方接口确认目标 `langgraph` 版本真实存在（例如 `https://pypi.org/pypi/langgraph/json` 的 `info.version`），并保证 Dockerfile `ARG VERSION`、`meta.yml` 版本键、目录名、README/image-info 链接四处版本号完全一致。

# CI 失败分析报告

## 基本信息
- PR: #4490 — 【自动升级】langgraph容器镜像升级至4.2.0版本.
- 失败类型: dependency-error
- 置信度: 高
- 知识库匹配: 模式02（软件包版本不存在）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#8 [3/3] RUN pip install --no-cache-dir -i https://pypi.tuna.tsinghua.edu.cn/simple langgraph==4.2.0
#8 0.669 Looking in indexes: https://pypi.tuna.tsinghua.edu.cn/simple
#8 1.176 ERROR: Ignored the following yanked versions: 0.0.8, 0.1.18, 0.2.29, ...
#8 1.176 ERROR: Could not find a version that satisfies the requirement langgraph==4.2.0
       (from versions: 0.0.9, ..., 1.2.10, 1.2.11, 1.2.12)
#8 1.176 ERROR: No matching distribution found for langgraph==4.2.0
#8 ERROR: process "/bin/sh -c pip install --no-cache-dir -i https://pypi.tuna.tsinghua.edu.cn/simple langgraph==${VERSION}" did not complete successfully: exit code: 1
...
ERROR: failed to solve: process "pip install ... langgraph==${VERSION}" did not complete successfully: exit code: 1
```
（日志末尾为 `Finished: FAILURE`，与失败状态一致，前置一致性检查通过。）

### 根因定位
- 失败位置: `AI/langgraph/4.2.0/24.03-lts-sp4/Dockerfile:11`
- 失败原因: Dockerfile 中 `ARG VERSION=4.2.0` 指定的 `langgraph==4.2.0` 在指定的 PyPI 索引上不存在；索引中可用的最高版本为 `1.2.12`，因此 pip 无法解析依赖并返回 exit code 1。

### 与 PR 变更的关联
直接相关。PR 新增 `AI/langgraph/4.2.0/24.03-lts-sp4/Dockerfile`，其中唯一的功能性构建步骤就是安装 `langgraph==${VERSION}`，而 `VERSION=4.2.0`。日志明确列出可选版本仅到 `1.2.12`，证明该版本号不存在（自动升级脚本抓取/解析到的版本号有误）。README.md、doc/image-info.yml、meta.yml 的改动仅为元数据登记，不构成失败原因。

## 修复方向

### 方向 1（置信度: 高）
将镜像版本（Dockerfile 目录名、`ARG VERSION`、`meta.yml`、`README.md`、`doc/image-info.yml` 中的 Tag 与链接）改为 PyPI 上真实存在的 langgraph 版本。日志显示当前可用最高版本为 `1.2.12`。需确认自动升级脚本为何从上游解析出 `4.2.0`——`image-info.yml` 中 `version_scheme: RPM` 与 `version_filter: cli;sdk` 的版本抓取规则可能误匹配了其他制品或非 langgraph 包的 tag，应核对上游 `langchain-ai/langgraph` 的实际发布版本号。

### 方向 2（可选）
若上游确实存在 4.2.0 但仅发布在官方 PyPI 而非清华镜像站，则应改用官方 `https://pypi.org/simple` 索引（或确认镜像站同步状态）后重试。但从日志中清华镜像返回的完整版本列表与 PyPI 公开版本一致看，该可能性较低。

## 需要进一步确认的点
- 确认上游 `langchain-ai/langgraph` 仓库是否存在 4.2.0 版本（日志证据已强烈表明不存在）。
- 核对 `AI/langgraph/doc/image-info.yml` 的 `version_filter: cli;sdk` 与 `version_scheme: RPM` 是否导致了错误的版本解析（4.2.0 疑似来自 cli/sdk 子包或其它 `version_filter` 命中项）。
- 确定应回退/更正到的实际版本号（当前可用最高 1.2.12），并同步更正所有引用该 Tag 的元数据文件。

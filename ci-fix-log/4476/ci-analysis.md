# CI 失败分析报告

## 基本信息
- PR: #4476 — 【自动升级】parquet容器镜像升级至1.18.1版本.
- 失败类型: dependency-error
- 置信度: 高
- 知识库匹配: 模式02（下载 URL 硬编码版本路径错误 / 软件包版本不存在）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#7 0.064   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
#7 0.923 curl: (22) The requested URL returned error: 404
#7 0.928 tar (child): parquet.tar.gz: Cannot open: No such file or directory
#7 0.928 tar (child): Error is not recoverable: exiting now
#7 0.929 tar: Child returned status 2
#7 0.929 tar: Error is not recoverable: exiting now
#7 ERROR: process "/bin/sh -c curl -fSL -o parquet.tar.gz https://archive.apache.org/dist/parquet/apache-parquet-format-${VERSION}/apache-parquet-format-${VERSION}.tar.gz; ..." did not complete successfully: exit code: 2
------
Dockerfile:6
   6 | >>> RUN curl -fSL -o parquet.tar.gz https://archive.apache.org/dist/parquet/apache-parquet-format-${VERSION}/apache-parquet-format-${VERSION}.tar.gz; \
ERROR: failed to solve: ... did not complete successfully: exit code: 2
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```
日志以 `Finished: FAILURE` 结束，为真实的下游 x86-64 构建 job 失败（非 trigger 层成功日志），前置一致性检查通过。

### 根因定位
- 失败位置: `Bigdata/parquet/1.18.1/24.03-lts-sp4/Dockerfile:6`
- 失败原因: 新增 Dockerfile 用 `${VERSION}=1.18.1` 拼出下载地址 `https://archive.apache.org/dist/parquet/apache-parquet-format-1.18.1/apache-parquet-format-1.18.1.tar.gz`，该地址在 `archive.apache.org` 返回 HTTP 404，制品不存在；`curl -f` 失败后 `parquet.tar.gz` 未生成，`tar` 随即报错，整个 RUN 退出码 2。

### 与 PR 变更的关联
完全由本次 PR 引入。PR 新增了 `Bigdata/parquet/1.18.1/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=1.18.1` 与第 6 行的 URL 模板组合出的下载路径不存在。同 PR 还向 `meta.yml`、`README.md`、`doc/image-info.yml` 注册了 `1.18.1-oe2403sp4`，使 CI 实际调度到该 Dockerfile 进行构建，从而触发失败。

补充说明：`doc/image-info.yml` 中 `upstream.version_prefix: apache-parquet-format` 表明自动升级工具按 `apache-parquet-format` 前缀识别版本。1.18.1 属于 parquet 的 1.x 版本线（parquet-mr 系列），与仓库中现有的 `apache-parquet-format-2.12.0 / 2.11.0`（parquet-format 2.x 版本线）不是同一制品命名。自动升级把 1.18.1 套用了 `apache-parquet-format-` 前缀，导致 URL 与真实制品不匹配。

## 修复方向

### 方向 1（置信度: 中）
修正下载地址/版本来源，使 `1.18.1` 指向真实存在的上游制品：
- 若 `1.18.1` 属 parquet-mr 版本线，则应将 URL 的制品名前缀由 `apache-parquet-format-` 改为对应制品名（如 `parquet-1.18.1` 之类），不能沿用 `apache-parquet-format-` 前缀。
- 或从上游镜像/归档站选取 `1.18.1` 实际发布的归档路径。
需先从 `archive.apache.org/dist/parquet/` 目录确认该版本真实制品名与目录名后再改。

### 方向 2（置信度: 中）
若上游并不存在 `1.18.1` 制品（自动升级误报版本），则应撤回该版本：删除 `Bigdata/parquet/1.18.1/` 目录，并同步移除 `meta.yml`、`README.md`、`doc/image-info.yml` 中的 `1.18.1-oe2403sp4` 条目。

## 需要进一步确认的点
1. `https://archive.apache.org/dist/parquet/` 下是否存在 1.18.1 对应的真实制品（确认目录名与文件名：是 `apache-parquet-format-1.18.1` 还是 `parquet-1.18.1` 或其他），这是判断走方向 1 还是方向 2 的关键。
2. `1.18.1` 究竟属于哪条上游版本线（parquet-format 2.x 还是 parquet-mr 1.x）；现有 `doc/image-info.yml` 的 `version_prefix: apache-parquet-format` 是否需要随版本线调整，否则后续自动升级会持续产生同类 404。
3. 新增的 4 个文件是否缺少模式17要求的 Copyright / SPDX 头（本次失败前未走到该检查，但可能成为后续校验失败点）。
4. 即便下载成功，第 12 行 `ENTRYPOINT [ "parquet", "--version" ]` 是否真有可执行文件 `/usr/local/parquet/bin/parquet`（parquet-format 为规范/源码仓，未必产出该二进制），存在后续运行期失败风险。

## 修复验证要求
本 PR 修复不涉及正则 patch 外部源文件，无该要求。但鉴于修复方向置信度为"中"，code-fixer 提交前必须：
1. 实际访问 `https://archive.apache.org/dist/parquet/`（或 `dlcdn.apache.org` / 华为云镜像站）确认 1.18.1 的真实制品名与完整可下载 URL，验证构造后的 URL 返回 200。
2. 若改为 parquet-mr 制品，确认该制品安装后确实提供 `parquet` 可执行文件，避免下载成功后 ENTRYPOINT/运行期失败。
3. 若判定版本不存在，确认删除 `1.18.1` 条目后各元数据文件（`meta.yml`、`README.md`、`doc/image-info.yml`）保持一致性。

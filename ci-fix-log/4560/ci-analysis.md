# CI 失败分析报告

## 基本信息
- PR: #4560 — 【自动升级】parquet容器镜像升级至1.18.1版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 模式02（下载 URL 硬编码版本路径错误 / 软件包版本不存在）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#7 [2/2] RUN curl -fSL -o parquet.tar.gz https://archive.apache.org/dist/parquet/apache-parquet-format-1.18.1/apache-parquet-format-1.18.1.tar.gz;     mkdir -p /usr/local/parquet &&     tar -zxf parquet.tar.gz ...
#7 0.809 curl: (22) The requested URL returned error: 404
#7 0.812 tar (child): parquet.tar.gz: Cannot open: No such file or directory
#7 0.812 tar (child): Error is not recoverable: exiting now
#7 ERROR: process "/bin/sh -c curl -fSL -o parquet.tar.gz https://archive.apache.org/dist/parquet/apache-parquet-format-${VERSION}/apache-parquet-format-${VERSION}.tar.gz; ..." did not complete successfully: exit code: 2
ERROR: failed to solve: process "... did not complete successfully: exit code: 2
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `Bigdata/parquet/1.18.1/24.03-lts-sp4/Dockerfile:6`（`RUN curl ...` 下载步骤）
- 失败原因: PR 新增 Dockerfile 中下载 `https://archive.apache.org/dist/parquet/apache-parquet-format-${VERSION}/apache-parquet-format-${VERSION}.tar.gz`，在 `VERSION=1.18.1` 展开后请求 `apache-parquet-format-1.18.1.tar.gz`，该归档在 `archive.apache.org` 上不存在，返回 HTTP 404；`curl -f` 立即失败，后续 `tar` 因文件不存在再次报错，整个 Docker 层以 exit code 2 失败。

### 与 PR 变更的关联
该失败由 PR 直接触发。PR diff 新增的 `Bigdata/parquet/1.18.1/24.03-lts-sp4/Dockerfile` 正是本次失败构建的对象，日志中的 404 URL 与 Dockerfile 第 6 行完全一致。同 PR 的 `README.md`、`doc/image-info.yml`、`meta.yml` 仅新增版本条目，日志显示 `The image specification check for releasing on appstore has passed.`（appstore 规范检查通过），说明失败仅发生在 Docker 构建阶段，与元数据文件无关。

补充疑点：现有 tag 为 `2.12.0`、`2.11.0`，而本次“升级”目标为 `1.18.1`，版本号反而更低；`doc/image-info.yml` 中 `upstream.version_prefix` 仍为 `apache-parquet-format`。日志证据只能确认该下载 URL 返回 404，无法确认 `1.18.1` 是否属于另一上游制品系列（如需确认见下文）。

## 修复方向

### 方向 1（置信度: 高）
修正新增 Dockerfile 的源码下载地址/版本，使其指向 `archive.apache.org/dist/parquet/` 下实际存在的归档。由于 `apache-parquet-format` 系列的版本与 `1.18.1` 可能并非同一制品系列，需先确认 `1.18.1` 对应的真实上游制品目录与文件名，再据以修正 `VERSION` 或下载 URL（不提供代码）。

### 方向 2（置信度: 中）
若自动升级脚本误取了错误的上游版本（`version_prefix: apache-parquet-format` 与实际取到的 `1.18.1` 不匹配，且低于现有 `2.12.0`），应修正（或重新生成）该版本的版本来源，使 `meta.yml` / `README.md` / `doc/image-info.yml` 与 Dockerfile 中的版本保持同一上游制品系列。

## 需要进一步确认的点
- 在 `archive.apache.org/dist/parquet/` 下确认是否存在 `apache-parquet-format-1.18.1/` 目录（当前日志显示 404，但需核对目录命名规则）。
- 确认 `1.18.1` 究竟属于哪个上游制品（`apache-parquet-format` 还是 parquet-java 等其他制品），从而判断应修改下载 URL 还是修改版本号。
- 核对自动升级工具为何选取 `1.18.1`（低于现有 `2.12.0`），以及 `doc/image-info.yml` 中 `version_prefix: apache-parquet-format` 是否与目标版本一致。
- 日志中未出现 aarch64 job 的失败信息，本次仅 x86-64 job 日志可确认；若两个架构均构建该 Dockerfile，arm64 应存在相同 404 问题。

## 修复验证要求
修复涉及修改外部下载 URL / 版本号时，code-fixer 在提交前必须验证目标 URL 实际可下载：以最终 Dockerfile 中 `VERSION` 的值为准，实际请求
`https://archive.apache.org/dist/parquet/apache-parquet-format-<VERSION>/apache-parquet-format-<VERSION>.tar.gz`
并确认返回 HTTP 200 且为有效 gzip 归档，验证通过后再提交，不能假设 404 版本之外的其他猜测版本一定可用。

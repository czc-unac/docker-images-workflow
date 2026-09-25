# CI 失败分析报告

## 基本信息
- PR: #4487 — 【自动升级】blat容器镜像升级至2.5.1版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式22
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#8 [3/6] RUN git clone -b 2.5.1 https://github.com/icebert/pblat-cluster.git /blat-cluster
#8 0.455 Cloning into '/blat-cluster'...
#8 1.161 fatal: Remote branch 2.5.1 not found in upstream origin
#8 ERROR: process "/bin/sh -c git clone -b ${VERSION} https://github.com/icebert/pblat-cluster.git /blat-cluster" did not complete successfully: exit code: 128
------
Dockerfile:9
   9 | >>> RUN git clone -b ${VERSION} https://github.com/icebert/pblat-cluster.git /blat-cluster
ERROR: failed to solve: process "/bin/sh -c git clone -b ${VERSION} https://github.com/icebert/pblat-cluster.git /blat-cluster" did not complete successfully: exit code: 128
```

### 根因定位
- 失败位置: `HPC/blat/2.5.1/24.03-lts-sp4/Dockerfile:9`（`RUN git clone -b ${VERSION} ...` 步骤）
- 失败原因: 新增 Dockerfile 中 `ARG VERSION=2.5.1`，`git clone -b 2.5.1` 尝试检出分支/标签 `2.5.1`，但上游仓库 `icebert/pblat-cluster` 中不存在名为 `2.5.1` 的 ref，Git 返回 `fatal: Remote branch 2.5.1 not found in upstream origin`（exit code 128），Docker 构建在 `[3/6]` 步骤即告失败。基础镜像 yum 安装步骤（`[2/6]`）已完成，错误发生在拉取源码阶段。

### 与 PR 变更的关联
高度相关且直接由本次 PR 触发。该 Dockerfile 为本次 PR 新增文件（`new_file: True`），其 `git clone -b ${VERSION}` 的版本分支名完全由新增内容决定。此前 PR 并未包含该镜像，因此失败在改动前不存在。日志中 `Cloning into '/blat-cluster'...` 紧接分支不存在的错误，说明仓库可达、仅 ref 名不匹配，非网络问题。

## 修复方向

### 方向 1（置信度: 高）
核实上游仓库 `icebert/pblat-cluster` 中 blat 2.5.1 对应的实际 ref 名称（分支或 tag），并据此修正 Dockerfile 中 `git clone -b` 使用的版本字符串。常见可能形式为 `v2.5.1`、`pblat-2.5.1` 或版本号带前缀/后缀。修正后 `ARG VERSION` 与 `-b` 引用应保持一致。

### 方向 2（置信度: 中）
若上游确实没有发布与 2.5.1 对应的分支/标签，则本次"自动升级"的目标版本号有误（自动升级脚本误判版本），应按上游实际发布的最高版本重做该镜像目录与 `meta.yml`、`README.md`、`doc/image-info.yml` 中的版本条目，而非仅改 clone 参数。

## 需要进一步确认的点
- 上游 `https://github.com/icebert/pblat-cluster` 中与 2.5.1 对应的确切分支名 / tag 名（需 `git ls-remote --tags --heads` 确认）。日志已证明 `2.5.1` 不存在，但未显示正确名称。
- 该版本实际是否存在（区分"ref 命名不一致"与"版本不存在"两种情况）。
- `doc/image-info.yml` 中 `version_scheme: RPM`、`upstream.version_url: icebert/blat-cluster` 的配置是否导致自动升级脚本推导出错误版本号（需确认版本发现逻辑是否会把 upstream tag 前缀剥离或拼接）。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
本次修复不涉及对第三方源文件的正则 patch，无需按该条验证。但 code-fixer 在提交前必须通过 `git ls-remote https://github.com/icebert/pblat-cluster` 确认目标 ref 实际存在，避免将另一个不存在的 ref 写入 Dockerfile。

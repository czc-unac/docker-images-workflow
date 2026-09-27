# CI 失败分析报告

## 基本信息
- PR: #4653 — 【自动升级】blat容器镜像升级至2.5.1版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式22（Git分支名构造错误，关键词高度吻合的“上游 ref 不存在”变体）
- 新模式标题: (不适用，命中已有模式)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#8 [3/6] RUN git clone -b 2.5.1 https://github.com/icebert/pblat-cluster.git /blat-cluster
#8 0.115 Cloning into '/blat-cluster'...
#8 0.651 fatal: Remote branch 2.5.1 not found in upstream origin
#8 ERROR: process "/bin/sh -c git clone -b ${VERSION} https://github.com/icebert/pblat-cluster.git /blat-cluster" did not complete successfully: exit code: 128
------
Dockerfile:9
--------------------
   7 |             openmpi-devel && \
   8 |         yum clean all && rm -rf /var/cache/yum
   9 | >>> RUN git clone -b ${VERSION} https://github.com/icebert/pblat-cluster.git /blat-cluster
ERROR: failed to solve: ... exit code: 128
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `HPC/blat/2.5.1/24.03-lts-sp4/Dockerfile:9`
- 失败原因: Dockerfile 以 `ARG VERSION=2.5.1` 执行 `git clone -b ${VERSION} https://github.com/icebert/pblat-cluster.git`，但上游仓库 `icebert/pblat-cluster` 中不存在名为 `2.5.1` 的远程 branch/tag，`git clone` 以 exit code 128（`fatal: Remote branch 2.5.1 not found in upstream origin`）失败。

### 与 PR 变更的关联
该失败由本 PR 直接触发。PR 新增了 `HPC/blat/2.5.1/24.03-lts-sp4/Dockerfile`，其中第 9 行的 clone 分支模板 `${VERSION}` 被展开为 `2.5.1`。日志确认该版本 ref 在上游 origin 不存在，故新镜像构建必失败。基础镜像 yum 安装步骤（[2/6]）已成功完成（`#7 DONE 82.4s`），失败仅发生在 [3/6] 的 clone 步骤，与依赖、编译、架构均无关。

## 修复方向

### 方向 1（置信度: 高）
核对上游 `icebert/pblat-cluster` 仓库中与 2.5.1 对应的真实 ref 名称（branch 或 tag），将 `ARG VERSION` / clone 分支名改为上游实际存在的正确 ref。当前 `2.5.1` 很可能并非该上游仓库使用的版本命名格式（该仓库历史版本曾使用 `1.1` 等，需确认 2.5.1 对应的实际 tag/branch）。修改前必须先在上游仓库确认目标 ref 存在。

### 方向 2（置信度: 中）
若上游确实没有 2.5.1 对应的发布 ref，则该“自动升级”目标版本有误，应改为上游最新可用版本，或在升级脚本源头修正版本来源（version_filter / version_scheme 配置，见 `HPC/blat/doc/image-info.yml` 的 `upstream` 段）。

## 需要进一步确认的点
- 上游 `icebert/pblat-cluster` 仓库中 2.5.1 对应 ref 的真实名称（`git ls-remote https://github.com/icebert/pblat-cluster.git` 查看全部 branch/tag）。日志只能证明“`2.5.1` 不存在”，无法证明正确名称是什么。
- 该上游仓库的版本命名规范：`HPC/blat/doc/image-info.yml` 中 `version_scheme: RPM`、`version_prefix` 为空，需确认自动升级生成的版本号与上游实际 ref 是否匹配。
- 若上游无 2.5.1，需确认自动升级任务为何选出版本 2.5.1（是否存在版本源解析错误）。

## 修复验证要求
本修复不涉及正则 patch 第三方文件，无需上游文件正则匹配验证。但 code-fixer 在提交前必须执行以下验证：
1. 从上游确认目标 ref 已存在：`git ls-remote https://github.com/icebert/pblat-cluster.git` 中必须能查到 Dockerfile 中使用的版本 ref，再提交修改。
2. 若调整了版本号，需同步更新 `HPC/blat/meta.yml`、`HPC/blat/README.md`、`HPC/blat/doc/image-info.yml` 及目录路径 `HPC/blat/<version>/24.03-lts-sp4/`，保持四处一致。

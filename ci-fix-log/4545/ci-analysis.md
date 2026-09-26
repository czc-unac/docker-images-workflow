# CI 失败分析报告

## 基本信息
- PR: #4545 — 【自动升级】ranger容器镜像升级至2.9.0版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式22（Git 分支名/ref 非法，症状关键词完全吻合）
- 新模式标题: （不适用）
- 新模式症状关键词: （不适用）

## 根因分析

### 直接错误
```
#8 [3/3] RUN ln -s /usr/bin/python3 /usr/bin/python &&     git clone -b v2.9.0 https://github.com/ranger/ranger.git && ...
#8 0.110 Cloning into 'ranger'...
#8 0.774 fatal: Remote branch v2.9.0 not found in upstream origin
#8 ERROR: process "/bin/sh -c ln -s /usr/bin/python3 /usr/bin/python &&     git clone -b v${VERSION} https://github.com/ranger/ranger.git ..." did not complete successfully: exit code: 128
------
Dockerfile:9
```

### 根因定位
- 失败位置: `Bigdata/ranger/2.9.0/24.03-lts-sp4/Dockerfile:10`（`git clone -b v${VERSION}` 步骤，第 9-14 行 RUN）
- 失败原因: Dockerfile 中 `ARG VERSION=2.9.0`，`git clone -b v${VERSION}` 展开为 `git clone -b v2.9.0 https://github.com/ranger/ranger.git`，但上游仓库 `ranger/ranger` 中不存在名为 `v2.9.0` 的远程分支/tag，git clone 以 exit code 128 失败。前置的 `yum install`（步骤 #7）已 `Complete!`，说明失败与基础依赖无关，唯一失败点就是第 3 个 RUN 的 clone。

### 与 PR 变更的关联
本 PR 为新增版本目录，新增文件 `Bigdata/ranger/2.9.0/24.03-lts-sp4/Dockerfile` 中引入了 `ARG VERSION=2.9.0`，CI 直接构建该新增 Dockerfile，因此该失败**由本次 PR 改动直接触发**。同 PR 的 `meta.yml`、`README.md`、`doc/image-info.yml` 改动只是登记新版本，不构成失败原因。

另一个值得注意的内部不一致：`Bigdata/ranger/doc/image-info.yml` 中 `upstream.version_url: apache/ranger`、`version_prefix: release-`，而 Dockerfile 实际克隆的是 `ranger/ranger` 且使用 `v` 前缀。版本探测来源（apache/ranger）与实际构建仓库（ranger/ranger）不一致，很可能是自动升级脚本据此取到了上游并不存在的 `2.9.0`。

## 修复方向

### 方向 1（置信度: 高）
修正新增 Dockerfile 中的版本号，使其与上游 `https://github.com/ranger/ranger` 实际存在的 tag/branch 一致。由于 `v2.9.0` 在该仓库不存在，应从上游仓库确认可用的实际 tag（该文件管理器历史版本为 1.9.x 系列），将 `VERSION` 及目录名改为真实存在的版本后重新提交。此方向不改变镜像语义。

### 方向 2（置信度: 中）
若该镜像的预期上游本应为 `apache/ranger`（见 image-info.yml 的 `version_url`），则当前 Dockerfile 克隆的仓库和 tag 前缀（`v`）均与声明的上游不匹配，需统一：明确目标项目后，修正 clone 的仓库地址与 tag 前缀（image-info.yml 声明前缀为 `release-`），并重新核对版本探测逻辑，避免自动升级再次取到不存在的版本。

## 需要进一步确认的点
1. `https://github.com/ranger/ranger` 中实际存在哪些可用的 tag/branch（尤其是与 2.9.0 对应的真实版本号），需据上游确认后再定版本。
2. 该镜像的语义究竟对应 `ranger/ranger`（终端文件管理器，历史版本 1.9.x）还是 `apache/ranger`（安全组件）——两者版本体系不同，决定采用方向 1 还是方向 2。
3. 自动升级脚本如何得出 `2.9.0`（是否有版本探测来源配置错误），需确认 `image-info.yml` 的 `version_url/version_prefix` 与实际构建仓库是否应保持一致。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
不适用（本次为 Dockerfile 自身版本参数错误，未涉及对外部源文件的正则 patch）。

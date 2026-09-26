# CI 失败分析报告

## 基本信息
- PR: #4535 — 【自动升级】cps_public容器镜像升级至5.2.5版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式22（Git 分支/标签名不存在，症状关键词完全命中）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#9 0.065 Cloning into 'CPS_public'...
#9 0.585 fatal: Remote branch v5.2.5 not found in upstream origin
#9 ERROR: process "/bin/sh -c git clone --depth 1 --branch v${VERSION} https://github.com/RBC-UKQCD/CPS_public.git && ... make && cp cps.a /usr/local/lib/ && cp -r ../CPS_public/cps_pp/include /usr/local/include/cps" did not complete successfully: exit code: 128
0.585 fatal: Remote branch v5.2.5 not found in upstream origin
ERROR: failed to solve: process "... git clone --depth 1 --branch v${VERSION} ..." did not complete successfully: exit code: 128
Dockerfile:13
  13 | >>> RUN git clone --depth 1 --branch v${VERSION} https://github.com/RBC-UKQCD/CPS_public.git && \
```

### 根因定位
- 失败位置: `HPC/cps_public/5.2.5/24.03-lts-sp4/Dockerfile:13`
- 失败原因: Dockerfile 用 `git clone --depth 1 --branch v${VERSION}` 克隆 `RBC-UKQCD/CPS_public`，`ARG VERSION=5.2.5` 展开后 ref 为 `v5.2.5`，但上游仓库不存在该分支/标签，git 返回 `Remote branch v5.2.5 not found in upstream origin`，退出码 128（依赖安装步骤 `[3/5] WORKDIR /build` 之前的 dnf 安装已成功，失败发生在 `[4/5]` 克隆步骤）。

### 与 PR 变更的关联
本 PR 直接新增 `HPC/cps_public/5.2.5/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=5.2.5` 与 `--branch v${VERSION}` 组合出的 ref `v5.2.5` 在上游不存在。同一 PR 中已存在的类似镜像条目使用的是目录/tag 形式 `5_2_5`（如 `5_2_5-oe2403sp4`、`5_2_5-oe2403sp3`，README 中标注 "CPS_public 5_2_5"），说明该上游仓库的版本标识使用下划线 `5_2_5`，而非点号 `5.2.5`。新增的 `v5.2.5` ref 与上游实际 ref 命名不一致，是本 PR 改动直接触发的失败。

## 修复方向

### 方向 1（置信度: 高）
修正新增 Dockerfile 中构造的 git ref，使其匹配上游 `RBC-UKQCD/CPS_public` 实际存在的标签/分支（对比同仓库中已成功构建的 `5_2_5` 相关 Dockerfile 的 ref 形式，判断应为下划线形式，如 `5_2_5` 或 `v5_2_5`，而非 `v5.2.5`）。同时同步核对新目录名、`meta.yml` 中 `5.2.5-oe2403sp4` 的 path、README/image-info.yml 的 tag 命名是否需要与上游版本标识保持一致。

### 方向 2（置信度: 中）
若上游确实以 `v5.2.5` 形式发布，则需确认该 release/tag 是否已推送（可能版本号尚未在 GitHub 打 tag），应以实际上游 tag 为准调整 `ARG VERSION`，或改用 commit/tag 的精确标识。

## 需要进一步确认的点
1. 上游 `https://github.com/RBC-UKQCD/CPS_public` 实际存在的 tag 列表，确认 `5.2.5` 对应的准确 ref（`v5_2_5` / `5_2_5` / 其他）。
2. 同仓库 `HPC/cps_public/5_2_5/24.03-lts-sp4/Dockerfile` 与 `5_2_5/24.03-lts-sp3/Dockerfile` 中 `git clone` 使用的 `--branch` 形式，作为命名参照。
3. 新增文件（Dockerfile/README/meta.yml/image-info.yml）是否缺少 Copyright + SPDX 头（模式17），但这不是本次构建失败的直接原因，属附带检查项。

## 修复验证要求
本修复不涉及对第三方源文件的正则 patch，无需按正则验证流程执行。但 code-fixer 在提交前应确认：修改后的 `--branch` ref 经 `git ls-remote https://github.com/RBC-UKQCD/CPS_public` 或等价方式核实确实存在于上游仓库，避免再次出现 `Remote branch ... not found in upstream origin`（exit code 128）。

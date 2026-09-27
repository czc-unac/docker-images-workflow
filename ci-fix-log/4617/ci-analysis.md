# CI 失败分析报告

## 基本信息
- PR: #4617 — 【自动升级】cps_public容器镜像升级至5.2.5版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式22（症状匹配：`fatal: Remote branch ... not found in upstream origin` + `exit code: 128`；但根因为“上游 tag 不存在”，而非模式22 的分支名重复拼接）

## 根因分析

### 直接错误
```
#9 0.058 Cloning into 'CPS_public'...
#9 0.577 fatal: Remote branch v5.2.5 not found in upstream origin
#9 ERROR: process "/bin/sh -c git clone --depth 1 --branch v${VERSION} https://github.com/RBC-UKQCD/CPS_public.git && ..." did not complete successfully: exit code: 128
ERROR: failed to solve: process ... exit code: 128
Dockerfile:13
```

### 根因定位
- 失败位置: `HPC/cps_public/5.2.5/24.03-lts-sp4/Dockerfile:13`
- 失败原因: 新增 Dockerfile 中 `git clone --branch v${VERSION}` 在 `VERSION=5.2.5` 下构造出的 ref `v5.2.5` 在 `github.com/RBC-UKQCD/CPS_public` 上游仓库中不存在，git 直接以 exit code 128 失败，导致 Docker 构建中断。

### 与 PR 变更的关联
本 PR 新增了 `HPC/cps_public/5.2.5/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=5.2.5` 且 clone 时使用 `--branch v${VERSION}`，失败 line 13 正是该新增文件的新增内容，属本次 PR 直接引入。仓库中已有的旧镜像目录为 `5_2_5`（下划线命名），说明上游 tag 命名与本次“点号版本号”可能不一致，进一步佐证 tag `v5.2.5` 不存在。

> 注：日志中 `perl-Storable` 的 `Curl error (56)` 为镜像站偶发网络抖动，RPM 随后安装成功（`Complete!`），**非**本次失败根因。

## 修复方向

### 方向 1（置信度: 高）
确认 `RBC-UKQCD/CPS_public` 上游仓库中与 5.2.5 对应的实际 tag/branch 命名，并据此修正 Dockerfile 中 `VERSION` 的值或 clone 的 ref 拼接方式（`--branch v${VERSION}`）。若上游 tag 采用下划线形式（如 `v5_2_5`，与现有 `5_2_5` 目录一致），则需调整 VERSION 或分支模板。

### 方向 2（置信度: 中）
若上游根本未发布 5.2.5 对应 tag，则本次“自动升级”目标版本不可构建，应撤回/关闭该新增镜像条目，避免引入必然失败的构建。

## 需要进一步确认的点
- `github.com/RBC-UKQCD/CPS_public` 上游仓库中 5.2.5 的实际 tag 名称（是 `v5.2.5`、`v5_2_5` 还是其他形式）；现有 `HPC/cps_public/5_2_5/24.03-lts-sp3`、`5_2_5/24.03-lts-sp4` 的 Dockerfile 中 `ARG VERSION` 实际取值及 clone 拼接方式可作对照。
- `HPC/cps_public/doc/image-info.yml` 中 `version_scheme: RPM` 与 `version_prefix: v` 的规则是否与上游实际 tag 匹配。
- 确认上游是否存在 5.2.5 发行版本；若不存在，则不是“tag 写法”问题而是版本本身不可用。

## 修复验证要求
若修复方向涉及修正 clone ref / VERSION 以匹配上游 tag：code-fixer 在提交前必须从 `RBC-UKQCD/CPS_public` 上游仓库确认 5.2.5 对应的真实 tag 名称（如通过 `git ls-remote --tags https://github.com/RBC-UKQCD/CPS_public.git`），验证修改后的 ref 确实存在后再提交，不得仅凭目录命名推断。

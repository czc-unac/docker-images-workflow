# CI 失败分析报告

## 基本信息
- PR: #4649 — 【自动升级】alluxio容器镜像升级至2.9.6版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式02
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#13 [7/8] RUN curl -fSL -o /tmp/alluxio.tar.gz https://downloads.alluxio.io/downloads/files/2.9.6/alluxio-2.9.6-bin.tar.gz
#13 0.680 curl: (22) The requested URL returned error: 404
#13 ERROR: process "/bin/sh -c curl -fSL -o /tmp/alluxio.tar.gz https://downloads.alluxio.io/downloads/files/${VERSION}/alluxio-${VERSION}-bin.tar.gz" did not complete successfully: exit code: 22
------
Dockerfile:16
--------------------
  14 |     WORKDIR ${ALLUXIO_HOME}
  15 |     RUN yum install -y java-1.8.0-openjdk-devel hostname
  16 | >>> RUN curl -fSL -o /tmp/alluxio.tar.gz https://downloads.alluxio.io/downloads/files/${VERSION}/alluxio-${VERSION}-bin.tar.gz
--------------------
ERROR: failed to solve: ... exit code: 22
Finished: FAILURE
```
（日志末尾为 `Finished: FAILURE`，与 PR 的失败状态一致，因此本次失败为真实构建失败，不属于"日志成功但状态失败"的证据不足场景。）

### 根因定位
- 失败位置: `Storage/alluxio/2.9.6/24.03-lts-sp4/Dockerfile:16`
- 失败原因: Dockerfile 中 `ARG VERSION="2.9.6"` 拼出的下载 URL `https://downloads.alluxio.io/downloads/files/2.9.6/alluxio-2.9.6-bin.tar.gz` 在上游返回 HTTP 404，即该版本二进制包在上游下载站不存在（版本号或下载路径不成立），curl 以退出码 22 终止，Docker build 失败。

### 与 PR 变更的关联
本 PR 为自动升级 PR，新增了 `Storage/alluxio/2.9.6/24.03-lts-sp4/Dockerfile`（`VERSION="2.9.6"`），并在 `meta.yml`、`image-info.yml`、`README.md` 中登记了 `2.9.6-oe2403sp4` 条目。失败发生在该新增 Dockerfile 的二进制下载步骤，与 PR 改动**直接相关**。升级脚本依据上游 tag（`Alluxio/alluxio`，`version_prefix: v`）认定 2.9.6 存在，但上游制品下载站并不存在对应的 `alluxio-2.9.6-bin.tar.gz`，属于"版本存在但制品不存在"的典型 404。

## 修复方向

### 方向 1（置信度: 高）
核对并修正下载源，使 URL 与上游实际发布的制品一致。可选做法（择一）：
- 确认 Alluxio 2.9.6 的官方二进制包实际下载地址（版本目录名或文件名可能与 `alluxio-${VERSION}-bin.tar.gz` 模板不符），改正 URL。
- 若 2.9.6 无官方二进制包，则改用上游确实提供二进制包的最近可用版本（如 2.9.5/2.9.4），并同步更新 `meta.yml`、`image-info.yml`、`README.md` 中登记的版本条目，保持仓库元数据一致。

### 方向 2（置信度: 中）
若确认 `downloads.alluxio.io` 对 CI 网络返回 404 系上游 CDN 路径变更（而非版本不存在），则将下载源切换为可靠镜像/归档源，或按模式01/33 的方式改用可稳定访问的归档地址。该方向需先验证目标地址真实可达。

## 需要进一步确认的点
- 需确认 Alluxio 2.9.6 官方是否发布了 `alluxio-2.9.6-bin.tar.gz`，以及其真实下载 URL；建议核对上游 release 页面与 `downloads.alluxio.io/downloads/files/2.9.6/` 目录实际内容。
- 需确认升级来源（upstream `Alluxio/alluxio` 的 tag `v2.9.6`）对应的制品命名规则，判断是"版本不存在"还是"下载路径模板错误"。
- 若替换为其他可用版本，需同步确认 `meta.yml` / `image-info.yml` / `README.md` 登记信息无误，避免元数据不一致引发后续校验失败。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
不适用（本失败为下载 URL/版本可用性问题，不涉及对第三方源文件的正则 patch）。

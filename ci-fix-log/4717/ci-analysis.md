# CI 失败分析报告

## 基本信息
- PR: #4717 — 【自动升级】qemu容器镜像升级至11.1.2版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式02（下载 URL 硬编码版本路径错误 / 软件包版本不存在）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#9 [4/6] RUN wget https://download.qemu.org/qemu-11.1.2.tar.xz     && tar xvJf qemu-11.1.2.tar.xz     && rm -f qemu-11.1.2.tar.xz
#9 0.062 --2026-09-29 06:29:50--  https://download.qemu.org/qemu-11.1.2.tar.xz
#9 0.091 Resolving download.qemu.org (download.qemu.org)... 79.127.235.3, 79.127.235.5, 79.127.235.8, ...
#9 0.148 Connecting to download.qemu.org (download.qemu.org)|79.127.235.3|:443... connected.
#9 0.210 HTTP request sent, awaiting response... 404 Not Found
#9 0.697 2026-09-29 06:29:50 ERROR 404: Not Found.
#9 ERROR: process "/bin/sh -c wget https://download.qemu.org/qemu-${VERSION}.tar.xz     && tar xvJf qemu-${VERSION}.tar.xz     && rm -f qemu-${VERSION}.tar.xz" did not complete successfully: exit code: 8
```

### 根因定位
- 失败位置: `Cloud/qemu/11.1.2/24.03-lts-sp4/Dockerfile:22`（下载步骤），版本变量定义于 `Dockerfile:4`（`ARG VERSION=11.1.2`）
- 失败原因: 构建时从 `https://download.qemu.org/qemu-11.1.2.tar.xz` 下载源码包返回 HTTP 404，文件在上游下载站不存在，`wget` 以 exit code 8 退出，Docker 构建中断。

### 与 PR 变更的关联
该 PR 为新增文件（`new_file: True`），新增了 `Cloud/qemu/11.1.2/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=11.1.2` 直接决定了下载 URL 中的版本号。此版本号对应的源码包在上游下载站不可用，因此本失败由本次 PR 引入的新版本号直接触发，与代码改动强相关（并非基础设施问题：日志显示已成功解析并连接 `download.qemu.org`，服务端明确返回 404，而非网络超时/不可达）。

### 非根因说明
- 日志末尾的 `JSONArgsRecommended: JSON arguments recommended for CMD ... (line 32)` 仅为 Docker 构建的 lint 警告，不是失败原因，不应作为修复依据。
- `dnf install` 步骤已成功完成（`#7 DONE 192.9s`），依赖安装无问题。

## 修复方向

### 方向 1（置信度: 高）
确认 QEMU 目标版本在上游下载站的真实可用性，并将 `VERSION` 调整为实际已发布且可下载的版本号（含正确的三位/两位版本号格式）。若 11.1.2 尚未正式发布或下载站不提供该文件，应回退到确认可用的版本（如沿用此前 11.1.1），或等待/切换到该版本的官方下载路径。

### 方向 2（置信度: 中）
若确认 11.1.2 确实已发布但不在 `download.qemu.org/qemu-11.1.2.tar.xz`，则需改用官方实际发布该制品的下载源/路径（如 QEMU 官网归档路径或对应 Release 制品地址），并同步更新下载文件名与解压命令以保持一致。

## 需要进一步确认的点
- 上游是否真实存在 `qemu-11.1.2` 版本（核对 QEMU 官方发布公告与 `download.qemu.org` 目录列表）；若为自动升级脚本误取未发布/占位版本，需要确认真实可升级到的版本号。
- `qemu-11.1.2.tar.xz` 的正确可下载 URL（确认官方是否提供该确切文件名与路径，避免与 QEMU GitHub Release 自动生成的 `qemu-11.1.2.tar.gz` 混淆）。
- 下载文件名与 `tar xvJf`（xz 格式）是否在替换后仍匹配（若改为 `.tar.gz` 则需相应改为 `tar xvzf`）。
- 若为自动升级流水线生成的版本号，确认升级来源（版本探测接口/上游 tag）与官方制品站点是否一致。

## 修复验证要求
本次失败不涉及"正则 patch 外部源文件"，故不需要正则匹配验证。
但 code-fixer 在提交前必须验证：目标版本号在 `download.qemu.org` 上确实可下载（以 HTTP 200 返回该 tarball），且下载文件名与解压命令格式（`.tar.xz` / `tar xvJf`）匹配，不能仅凭推断假定版本可用。

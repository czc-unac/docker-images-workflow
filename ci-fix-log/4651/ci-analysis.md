# CI 失败分析报告

## 基本信息
- PR: #4651 — 【自动升级】libvirt容器镜像升级至2026.77159版本.
- 失败类型: dependency-error
- 置信度: 高
- 知识库匹配: 模式02
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#9 [4/6] RUN wget https://download.libvirt.org/libvirt-2026.77159.tar.xz \
    && tar -xvf libvirt-2026.77159.tar.xz \
    && rm -f libvirt-2026.77159.tar.xz
#9 0.065 --2026-09-27 09:08:08--  https://download.libvirt.org/libvirt-2026.77159.tar.xz
#9 0.114 Resolving download.libvirt.org (download.libvirt.org)... 149.202.84.84
#9 0.135 Connecting to download.libvirt.org (download.libvirt.org)|149.202.84.84|:443... connected.
#9 0.848 HTTP request sent, awaiting response... 404 Not Found
#9 1.202 2026-09-27 09:08:09 ERROR 404: Not Found.
#9 ERROR: process "/bin/sh -c wget https://download.libvirt.org/libvirt-${VERSION}.tar.xz ..." did not complete successfully: exit code: 8
ERROR: failed to solve: process "... wget ... libvirt-${VERSION}.tar.xz ..." did not complete successfully: exit code: 8
Dockerfile:16
```

### 根因定位
- 失败位置: `Cloud/libvirt/2026.77159/24.03-lts-sp4/Dockerfile:16-18`（`RUN wget https://download.libvirt.org/libvirt-${VERSION}.tar.xz ...` 步骤）
- 失败原因: Dockerfile 中 `ARG VERSION=2026.77159` 指向的 libvirt 源码包在 `https://download.libvirt.org/libvirt-2026.77159.tar.xz` 不存在（HTTP 404），wget 以 exit code 8 退出导致 Docker 构建失败。

### 与 PR 变更的关联
本 PR 为自动升级，新增 `Cloud/libvirt/2026.77159/24.03-lts-sp4/Dockerfile`，其中硬编码 `ARG VERSION=2026.77159`，并在第 16 行据此拼接下载 URL。该版本号并非 libvirt 官方实际发布的版本（仓库既有版本为 `12.7.0`、`12.4.0` 这类 `主.次.修订` 格式，而非 `2026.77159` 这种年份+序号格式），因此上游 `download.libvirt.org` 无对应制品，构建必然 404。失败由本 PR 新增内容直接触发。

补充：日志 `#7` 阶段的 `depmod: ERROR`/`dpdk ... POSTTRANS scriptlet failed` 仅为安装脚本的 warning，后续 255 个包全部 `Verifying`/`Installed` 成功且该层 `#7 DONE 105.3s`，非失败根因。

## 修复方向

### 方向 1（置信度: 高）
确认 libvirt 官方实际存在且可下载的版本号，将 `Cloud/libvirt/2026.77159/.../Dockerfile` 中的 `ARG VERSION` 及其所在版本目录名、README/image-info.yml/meta.yml 中的版本引用统一修正为真实存在的 libvirt 版本（需与 `download.libvirt.org` 上的文件名一致）。

### 方向 2（置信度: 中）
若该版本确实由上游发布但其归档路径发生变化，则将下载源切换为保留历史版本的官方归档/镜像站路径，并按实际文件名修正 VERSION 拼接。

## 需要进一步确认的点
- `2026.77159` 是否为 libvirt 官方真实发布版本号？需核对 `https://download.libvirt.org/` 目录中实际存在的 tar.xz 文件名。
- 若为自动升级脚本误判版本（如把日期/构建号当作版本），需确认正确的上游版本号来源。
- 既有可参考版本 `12.7.0` 的 Dockerfile 其下载 URL 是否可正常访问，以确认下载源本身可用。

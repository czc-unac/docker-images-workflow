# CI 失败分析报告

## 基本信息
- PR: #4485 — 【自动升级】libvirt容器镜像升级至2026.77159版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式02（下载 URL 硬编码版本路径错误 / 软件包版本不存在）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#9 [4/6] RUN wget https://download.libvirt.org/libvirt-2026.77159.tar.xz     && tar -xvf libvirt-2026.77159.tar.xz     && rm -f libvirt-2026.77159.tar.xz
#9 0.059 --2026-09-25 08:05:27--  https://download.libvirt.org/libvirt-2026.77159.tar.xz
#9 0.264 Connecting to download.libvirt.org (download.libvirt.org)|149.202.84.84|:443... connected.
#9 0.911 HTTP request sent, awaiting response... 404 Not Found
#9 1.234 2026-09-25 08:05:28 ERROR 404: Not Found.
#9 ERROR: process "/bin/sh -c wget https://download.libvirt.org/libvirt-${VERSION}.tar.xz ..." did not complete successfully: exit code: 8
...
Dockerfile:16
  16 | >>> RUN wget https://download.libvirt.org/libvirt-${VERSION}.tar.xz \
  17 | >>>     && tar -xvf libvirt-${VERSION}.tar.xz \
  18 | >>>     && rm -f libvirt-${VERSION}.tar.xz
ERROR: failed to solve: process ... exit code: 8
```

### 根因定位
- 失败位置: `Cloud/libvirt/2026.77159/24.03-lts-sp4/Dockerfile:16`（wget 下载源码步骤）
- 失败原因: 目标 URL `https://download.libvirt.org/libvirt-2026.77159.tar.xz` 返回 HTTP 404，即 libvirt `2026.77159` 这个版本在官方下载站不存在。wget 退出码 8 表示服务器返回错误响应（4xx），导致 Docker 构建终止。

### 与 PR 变更的关联
本 PR 为自动升级 PR，新增的 `Dockerfile` 中 `ARG VERSION=2026.77159`，并据此拼接下载 URL。404 直接由该新增版本号触发，与 PR 改动**直接相关**。

需要特别注意：libvirt 上游正常版本号形如 `12.7.0`、`12.4.0`（见仓库 README 中已有条目），而 `2026.77159` 这种“年份+纯数字”格式不符合 libvirt 官方版本命名规律，极可能是自动升级脚本抓取到了错误的版本/标签（例如误抓了日期或其它上游标识），导致下载地址根本不存在。

> 说明：日志中 `#7` 阶段的 `depmod: ERROR: could not open directory /lib/modules/...` 与 `%posttrans(dpdk...) scriptlet failed` 属于 dnf 安装 dpdk 时的常见非致命告警，该阶段最终输出 `#7 DONE 145.7s` 成功完成，**不是本次失败根因**。真正的致命错误是 `#9` 步骤的 404。

## 修复方向

### 方向 1（置信度: 高）
确认 libvirt 官方实际存在的最新版本号，将 `Dockerfile` 中的 `ARG VERSION`（以及 `README.md`、`doc/image-info.yml`、`meta.yml` 中对应的 `2026.77159` 标识与目录名）修正为上游真实可下载的版本。若自动升级流程抓取了非法版本，应对升级脚本/数据源进行校验（例如先探测 `download.libvirt.org` 上该版本 tarball 是否可访问）。

### 方向 2（可选）
若确实需要发布该版本而非上游 tarball，则改用其它可用的下载源或构建方式定位源码包。但当前证据显示该版本在官方站点不存在，此方向风险较高。

## 需要进一步确认的点
1. libvirt `2026.77159` 是否为真实存在的上游版本——需从 `https://download.libvirt.org/` 目录列表核实实际可用版本号。若不存在，本次自动升级的源数据/触发逻辑本身需要修正，而非仅改单个 Dockerfile。
2. 自动升级脚本从何处获得 `2026.77159` 这一版本串，是否存在版本解析错误（例如把日期 `2026.7...` 误判为版本）。
3. 若确定改用正确版本，需同步核对 README.md、doc/image-info.yml、meta.yml 以及目录路径 `Cloud/libvirt/<VERSION>/24.03-lts-sp4/` 的一致性。

## 修复验证要求
本次修复若涉及修改版本号，code-fixer 在提交前必须验证：从上游 `download.libvirt.org` 实际访问 `libvirt-<VERSION>.tar.xz` 返回 200（可下载），确认新版本号真实存在后再同步更新 Dockerfile、README.md、doc/image-info.yml、meta.yml 及目录名。若修复方案包含对自动升级数据源的修改，需验证升级脚本产出的版本号确为官方有效版本。

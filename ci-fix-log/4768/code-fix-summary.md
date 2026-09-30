# 修复摘要

## 修复的问题
自动升级 PR 把不存在的版本号 `234b974` 拼进 ISC DHCP 下载 URL，导致
`https://ftp.isc.org/isc/dhcp/234b974/dhcp-234b974.tar.gz` 返回 HTTP 404（wget exit code 8），
Docker 构建在 `Others/dhcp/234b974/24.03-lts-sp4/Dockerfile` 的 `wget` 步骤失败（x86_64、aarch64 均失败）。
已将其修正为 ISC DHCP 上游真实存在的发布版本 `4.4.3-P1`。

## 修改的文件
- `Others/dhcp/234b974/24.03-lts-sp4/Dockerfile`: 第 3 行 `ARG VERSION=234b974` → `ARG VERSION=4.4.3-P1`，使 `wget .../${VERSION}/dhcp-${VERSION}.tar.gz` 指向真实制品。
- `Others/dhcp/meta.yml`: 条目键 `234b974-oe2403sp4:` → `4.4.3-P1-oe2403sp4:`（`path` 保持 `234b974/24.03-lts-sp4/Dockerfile` 不变）。
- `Others/dhcp/doc/image-info.yml`: tags 表格首行 Tag 由 `234b974-oe2403sp4` 改为 `4.4.3-P1-oe2403sp4`，说明由 `dhcp 234b974` 改为 `DHCP 4.4.3-P1`（链接目录路径保持不变）。
- `Others/dhcp/README.md`: Supported tags 表格首行 Tag 与说明做同样修正。

## 修复逻辑
本报告原文置信度为「低」且未附日志。为消除歧义，本次未停留在 diff 推断，而是取得了**真实 CI 日志**并做了上游核实：

1. **真实 CI 结论（已获取）**：从 PR #4768 的门禁评论定位到构建日志（x86_64 build #4879 / aarch64 build #4975），
   通过 `https://log-ci.openeuler.openatom.cn/api/build/log/download?job=multiarch/openeuler/x86-64/openeuler-docker-images&build=4879`
   拉取日志，末尾明确报错：
   ```
   #8 RUN wget https://ftp.isc.org/isc/dhcp/234b974/dhcp-234b974.tar.gz ...
   404 Not Found
   ERROR 404: Not Found.
   #8 ERROR: process "... wget ..." did not complete successfully: exit code: 8
   Finished: FAILURE
   ```
   即根因确认为分析报告中「模式02（下载 URL 版本路径不存在）」，而非许可证问题。
   （门禁的 `check_package_license` 仅为 **WARNING**：缺少项目级 Copyright 声明文件，属仓库既有状态，不是本次构建失败的原因。）

2. **目标版本核实（已从上游验证）**：
   - `234b974` 并不存在于 ISC DHCP 的任何发布目录，也不是 `isc-projects/dhcp`（GitHub/GitLab）中的提交；
     实为 `insomniacslk/dhcp`（一个 Go 库）的提交，与基于 ISC DHCP 的该镜像无关，属自动升级取版本时取错了来源。
   - 上游真实最新发布版本为 **`4.4.3-P1`**：
     - `https://ftp.isc.org/isc/dhcp/` 目录中 `4.4.3-P1/` 存在（最后更新 2022-10-05）；
     - release-monitoring(anitya) 项目 `dhcp`(id 5584) 的当前版本为 `4.4.3-P1`；
     - openEuler 软件市场 `dhcp` 的 appversion 亦声明上游版本 `4.4.3-P1`（状态 MISSING）。
   - 已实际下载验证 tarball：`https://ftp.isc.org/isc/dhcp/4.4.3-P1/dhcp-4.4.3-P1.tar.gz` 返回 **HTTP 200**（约 10 MB），
     解压后顶层目录为 `dhcp-4.4.3-P1/`，其中包含 `configure`，与 Dockerfile 的
     `cd dhcp-${VERSION}` / `./configure` / `make -j$(nproc)` / `make install` 步骤完全匹配。

3. **改动范围最小化**：仅修正版本号及其在 README/meta.yml/image-info.yml 中的对外 Tag/说明，使其自洽。
   受「只能修改原始 PR 涉及文件、不允许新增文件」约束，版本目录 `234b974/24.03-lts-sp4/` 无法重命名，
   故保留目录路径不变，仅把对外版本标识修正为真实版本（与仓库历史同类修复做法一致，参见 PR #4651/#4649）。

## 潜在风险
- 镜像目录名 `234b974/` 与对外 Tag `4.4.3-P1-oe2403sp4` 不一致（目录名为自动升级误生成的既有历史值）。
  这是「不新增文件」约束下的必然结果；`meta.yml` 的 `path` 与 README/image-info 的链接均已指向同一 Dockerfile，CI 构建不受影响。
- `image-info.yml` 的 `upstream.regex` 仍为 `(\d+\.\d+\.\d+)`，无法捕获 `-P1` 后缀；若后续自动升级仍使用该字段，
  可能再次生成不带 `-P1` 的版本。此为上游配置项、非本次构建失败直接原因，按最小化原则未改动，建议作为后续跟进。
- 除此之外无功能风险：仅将下载目标从 404 的 `234b974` 换为真实存在且可解压构建的 `4.4.3-P1`。
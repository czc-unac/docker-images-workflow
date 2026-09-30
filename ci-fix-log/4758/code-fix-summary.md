# 修复摘要

## 修复的问题
新增的 `Others/glibc/2.42.9000` 镜像构建失败（x86_64/aarch64 均失败），根因是 `glibc-2.42.9000.tar.xz` 在下载源不存在（HTTP 404），将该开发快照版本改为其对应的正式发行版 `2.43`。

## 修改的文件
- `Others/glibc/2.42.9000/24.03-lts-sp4/Dockerfile`: `ARG VERSION=2.42.9000` → `ARG VERSION=2.43`
- `Others/glibc/meta.yml`: 条目键 `2.42.9000-oe2403sp4` → `2.43-oe2403sp4`（path 保持不变）
- `Others/glibc/README.md`: 表格行 tag 与描述由 `2.42.9000-oe2403sp4` / `glibc 2.42.9000` → `2.43-oe2403sp4` / `glibc 2.43`（Dockerfile 链接路径保持不变）
- `Others/glibc/doc/image-info.yml`: 同上表格行 tag 与描述更新为 `2.43`

## 修复逻辑
1. **证据确认（非猜测）**：`ci_analysis` 无日志，因此我从原始 PR #4758 的 CI 评论中定位到两个真实失败 job，并下载了完整控制台日志：
   - x86_64: `https://log-ci.openeuler.openatom.cn/api/build/log/download?job=multiarch/openeuler/x86-64/openeuler-docker-images&build=4869`
   - aarch64: `.../aarch64/openeuler-docker-images&build=4965`
   两架构均在 `Dockerfile:18` 的 `wget .../gnu/glibc/glibc-2.42.9000.tar.xz` 步骤失败，报 `HTTP request sent, awaiting response... 404 Not Found` / `ERROR 404: Not Found` / `exit code: 8`，即 **模式02（软件包版本不存在）**。
2. **实际访问 URL 验证**：`curl -I` 确认清华镜像 `mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-2.42.9000.tar.xz` 与 GNU 官方 `ftp.gnu.org/gnu/glibc/glibc-2.42.9000.tar.xz` 均返回 **404**；镜像目录中仅存在正式发行版 `glibc-2.42.tar.xz`、`glibc-2.43.tar.xz`、`glibc-2.44.tar.xz`。`.9000` 后缀是 glibc 的开发快照版本号（`glibc-2.42.9000` 是 2.42 发布后开启的 **2.43 开发周期** 的滚动 tag，仅存在于 git 仓库，不发布到 GNU release 镜像）。
3. **修复方式**：按本仓库自动修复的既有约定（参考 `Database/redis/5.4.1`、`Storage/alluxio/2.9.6`、`Others/mongoose/7.22` 的历史 auto-fix：只把 `ARG VERSION` 换成实际可下载的版本，目录路径保持不变，并同步更新 meta/README/image-info 的版本标签），将该开发版本替换为其对应的正式发行版 **2.43**，并同步三处元数据，使对外 tag 与实际构建内容一致。
4. **可达性验证**：`curl -I https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-2.43.tar.xz` 返回 **HTTP 200**，下载链接有效，`Dockerfile` 其余步骤（多阶段构建、`--disable-werror`）与已验证可用的 2.42 版本一致，无需改动。

## 潜在风险
- 目录路径仍为 `2.42.9000/24.03-lts-sp4` 而实际构建 2.43，存在路径与版本号字面不一致；这与本仓库历史 auto-fix（redis/alluxio/mongoose）的做法一致，属于既有约定，不影响构建与镜像 tag。
- 2.43 相比 2.42 的 glibc 升级幅度较小（同一开发周期对应的正式版），构建工具链要求与 2.42 基本一致（SP4 自带 gcc 12.3/make 4.4 满足要求），并沿用 `--disable-werror` 规避 SP4 内核头文件的宏重定义告警，预期构建可通过。
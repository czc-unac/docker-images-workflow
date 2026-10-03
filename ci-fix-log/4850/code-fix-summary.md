# 修复摘要

## 修复的问题
新增的 `Others/glibc/2.42.9000` 镜像在 x86_64/aarch64 构建时，`wget https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-2.42.9000.tar.xz` 返回 HTTP 404，导致 Docker 构建失败。根因是该 `.9000` 后缀为 glibc 开发快照版本号，GNU release 镜像站不发布对应 tarball；将该版本修正为其对应的正式发行版 `2.43`。

## 修改的文件
- `Others/glibc/2.42.9000/24.03-lts-sp4/Dockerfile`: `ARG VERSION=2.42.9000` → `ARG VERSION=2.43`（下载 URL、解压目录、`WORKDIR /opt/glibc-${VERSION}/build` 均随变量生效，无需其他改动）
- `Others/glibc/meta.yml`: 条目键 `2.42.9000-oe2403sp4` → `2.43-oe2403sp4`（`path` 保持指向 `2.42.9000/24.03-lts-sp4/Dockerfile`）
- `Others/glibc/README.md`: 表格行 tag/描述由 `2.42.9000-oe2403sp4` / `glibc 2.42.9000` → `2.43-oe2403sp4` / `glibc 2.43`（Dockerfile 链接路径保持不变）
- `Others/glibc/doc/image-info.yml`: 同上表格行 tag/描述更新为 `2.43`

## 修复逻辑
1. **证据确认（来自真实 CI 日志，非猜测）**：虽然入参 `ci_analysis` 标注日志缺失、置信度低，我从原始 PR #4850 的 CI 评论中定位到两个失败 job 并下载了控制台日志：
   - x86_64: `https://log-ci.openeuler.openatom.cn/api/build/log?job=multiarch/openeuler/x86-64/openeuler-docker-images&build=4964`
   - aarch64: `https://log-ci.openeuler.openatom.cn/api/build/log?job=multiarch/openeuler/aarch64/openeuler-docker-images&build=5060`
   两架构均在 `Dockerfile:18` 的 `RUN wget .../gnu/glibc/glibc-2.42.9000.tar.xz` 步骤失败，报 `HTTP request sent, awaiting response... 404 Not Found` / `ERROR 404: Not Found.` / `exit code: 8`（错误原文：`process "/bin/sh -c wget https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-${VERSION}.tar.xz ..." did not complete successfully`）。经 `docker build` 展开后 URL 为 `glibc-2.42.9000.tar.xz`，即**模式02（下载 URL / 软件包版本不存在）**。
2. **URL 实际访问验证**：`curl` 确认 `https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-2.42.9000.tar.xz` 返回 **404**，GNU 官方 `https://ftp.gnu.org/gnu/glibc/glibc-2.42.9000.tar.xz` 亦返回 **404**；镜像目录中仅有正式发行版 `glibc-2.42/2.43/2.44.tar.xz`。`.9000` 是 glibc 开发周期滚动版本号，只存在于 git/alpha 快照，不发布到 GNU release 镜像站。
3. **修复方式**：遵循本仓库自动修复的既有约定（历史 auto-fix：`Database/redis/5.4.1`、`Others/dnsmasq/4.3.1`、`Others/binder/3.0` 等均只将 `ARG VERSION` 换为实际可下载版本、目录路径保持不变并同步元数据；同仓库上一轮 glibc PR #4758 的同类问题亦采用同一处理），将开发版本替换为其对应正式发行版 **2.43**，并同步 `meta.yml`、`README.md`、`image-info.yml` 三处标签，确保对外 tag 与实际构建内容一致。
4. **可达性验证**：`curl -I https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-2.43.tar.xz` 返回 **HTTP 200**（`content-type: application/octet-stream`），目标下载链接有效。Dockerfile 其余步骤（多阶段构建、`--disable-werror` 规避 SP4 内核头文件宏重定义告警）与已验证可用的 2.42/2.43 版本一致，无需改动。

## 潜在风险
- 目录路径仍为 `2.42.9000/24.03-lts-sp4` 而实际构建 `2.43`，存在路径与版本字面不一致；这与本仓库历史 auto-fix 的既有约定一致，不影响构建与镜像 tag，且避免了创建/重命名新目录（受限于只允许修改原 PR 涉及的文件）。
- `2.43` 在 GNU 镜像站与 `2.42` 同属相邻发行版，构建工具链要求（SP4 自带 gcc/make）基本一致，预期构建可通过；最终需由 CI 复核 x86-64 与 aarch64 两个架构。
- 本次修复不涉及对第三方/上游源文件的正则 patch，无需 `re.sub` 验证。
# 修复摘要

## 修复的问题
glibc 2.42.9000 镜像构建时下载的 `glibc-2.42.9000.tar.xz` 在上游/镜像站不存在（HTTP 404），导致 `wget` 失败、镜像构建失败；将 Dockerfile 中源码版本修正为实际发布的 `2.42`。

## 修改的文件
- `Others/glibc/2.42.9000/24.03-lts-sp4/Dockerfile`: `ARG VERSION` 由 `2.42.9000` 改为 `2.42`（第 3 行）。

## 修复逻辑
分析报告在无日志情况下给出三个候选假设（版本 404 / 缺构建依赖 / 缺 license 头）。逐一核实后定位为**假设 1（版本不存在导致下载 404）**，对应报告"方向 1 若为下载 404"：

1. **上游版本核实（已实测）**：
   - `curl -sI https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-2.42.9000.tar.xz` → `404 Not Found`；
   - `curl -sI https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-2.42.tar.xz` → `200 OK`（19,930,508 字节）；
   - 与 `image-info.yml` 声明的上游源一致，`https://ftp.gnu.org/gnu/glibc/` 目录列表中亦无 `glibc-2.42.9000.tar.xz`，只有 `2.41/2.42/2.43/2.44` 等正式发布版。
   - glibc 中 `X.YY.9000` 是开发分支内部版本号（2.42.9000 即 2.42 发布前的开发版），镜像站只提供正式发布 tarball，故该版本必然 404。修正为 `2.42` 即为该开发版对应的正式发布版本。
2. **排除假设 2（缺依赖）**：修改后的 Dockerfile 与仓库中已能正常构建的 `Others/glibc/2.42/24.03-lts-sp4/Dockerfile` 完全一致（同样的 `bison gcc gcc-c++ make wget xz` 依赖与 `--disable-werror` 配置），无需补装 `gawk/texinfo/gmp-devel/mpfr-devel/libmpc-devel`。
3. **排除假设 3（license 头）**：仓库内 glibc 全部 Dockerfile（2.41/2.42 等）均无 `Copyright/SPDX-License-Identifier` 头，CI 未对该目录强制 license 头检查。

因此仅修改下载所用的 `VERSION`，使构建与已有 2.42 镜像保持一致。

## 潜在风险
该 tag/目录名仍为 `2.42.9000`，而最终镜像内实际为 glibc 2.42，存在命名与内容不完全一致的情况（README.md / meta.yml / image-info.yml 中的 tag 名为 `2.42.9000-oe2403sp4`）。这是自动升级单采用开发版本号所致；不改动 tag 名是为保持改动最小、避免与已存在的 `2.42-oe2403sp4` 条目冲突。此命名差异不影响镜像构建与 CI 通过。
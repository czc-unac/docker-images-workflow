# 修复摘要

## 修复的问题
自动升级生成的版本号 `2.42.9000` 在上游 GNU glibc 并不存在，Dockerfile 下载源码 tarball 会返回 404 导致构建失败；将其改为上游真实发布版本 `2.42`。

## 修改的文件
- `Others/glibc/2.42.9000/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=2.42.9000` 改为 `ARG VERSION=2.42`。

## 修复逻辑
- 已从上游源（`image-info.yml` 声明的 `https://ftp.gnu.org/gnu/glibc/` 及 Dockerfile 实际使用的清华镜像 `https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/`）核对目录列表：其中只有 `glibc-2.40/2.41/2.42/2.43/2.44` 等发布制品，**不存在 `glibc-2.42.9000.tar.xz`**。实测 HTTP 状态：`glibc-2.42.9000.tar.xz` 返回 **404**，`glibc-2.42.tar.xz` 返回 **200**。
- `2.42.9000` 属于 glibc git 开发分支内部的开发版本号，并非上游发布的 tarball；自动升级工具误将其识别为新版本并据此建立目录/元数据，导致构建时 `wget` 下载 404。
- 修复后，新文件 `Others/glibc/2.42.9000/24.03-lts-sp4/Dockerfile` 与仓库中已验证可正常构建的 `Others/glibc/2.42/24.03-lts-sp4/Dockerfile` **逐字节完全一致**（经 `diff` 校验），依赖列表（bison/gcc/gcc-c++/make/wget/xz）沿用既有可通过构建的配置，故不再额外补装依赖，避免超出最小化修复范围。
- 该修复对应分析报告"候选假设 1：上游版本不存在导致下载 404"方向，并已按报告要求从声明上游源核对目标版本真实存在后再修改版本号。

## 潜在风险
- 镜像 tag 仍为 `2.42.9000-oe2403sp4`，而 Dockerfile 实际构建的是 glibc `2.42`，README/image-info.yml 中"glibc 2.42.9000"的文字描述与实际版本存在命名不一致。本次仅修复构建失败，未改动元数据以免与已有 `2.42-oe2403sp4` 条目冲突；如需消除该不一致，应由自动升级流程在版本过滤阶段排除 `.9000` 开发版本，而非在镜像仓中手工改名。
- 不影响 `2.42`、`2.41` 等既有版本文件的构建。
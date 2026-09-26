# 修复摘要

## 修复的问题
修复 guacd 1.6.0（openEuler 24.03-LTS-SP4）镜像构建时 guacamole-server configure 因基础镜像 FreeRDP 被识别为开发/快照版而报错退出的问题。

## 修改的文件
- `Others/guacd/1.6.0/24.03-lts-sp4/Dockerfile`: 在 `ARG GUACAMOLE_SERVER_OPTS` 中新增 `--enable-allow-freerdp-snapshots`（保留原有 `--disable-guaclog`）。

## 修复逻辑
CI 失败根因是 guacamole-server 1.6.0 的 configure 脚本检测到 openEuler 24.03-LTS-SP4 仓库安装的 FreeRDP 为 development/snapshot 版本，在未显式允许的情况下直接 `AC_MSG_ERROR` 退出。修复采用分析报告的“方向 1”：向 configure 传入其官方开关 `--enable-allow-freerdp-snapshots`，使构建继续进行。

已从上游 `apache/guacamole-server` 的 `1.6.0` tag 拉取 `configure.ac` 验证：其中确实存在 `AC_ARG_ENABLE(allow_freerdp_snapshots, ... [--enable-allow-freerdp-snapshots] ...)` 定义，且正是触发该报错的检查逻辑，选项名与 1.6.0 源码完全一致。

未对 `README.md`、`doc/image-info.yml`、`meta.yml` 做改动，这些文件仅包含版本/Tag 元数据，与本次构建失败无关。

## 潜在风险
低。该开关仅跳过 FreeRDP 开发版的版本号检查，不改变其它构建行为；guacamole-server 后续仍会针对 FreeRDP 各 API 差异做编译期特性探测。若基础镜像仓库后续改为提供正式版 FreeRDP，可再评估移除此开关。
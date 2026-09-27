# 修复摘要

## 修复的问题
guacd 1.6.0 (24.03-lts-sp4) 构建时，`./configure` 检测到基础镜像中的 FreeRDP 为开发版（development/snapshot）而报错退出，导致镜像构建失败。

## 修改的文件
- `Others/guacd/1.6.0/24.03-lts-sp4/Dockerfile`: 在 `ARG GUACAMOLE_SERVER_OPTS` 的配置项中，保留 `--disable-guaclog` 并追加 `--enable-allow-freerdp-snapshots`，使 configure 显式放行开发版 FreeRDP。

## 修复逻辑
- CI 日志中 `configure` 报错位置为 `Others/guacd/1.6.0/24.03-lts-sp4/Dockerfile:25`，由第 7-8 行的 `GUACAMOLE_SERVER_OPTS` 传入选项控制。原选项仅含 `--disable-guaclog`，未放行开发版 FreeRDP，故 configure 在 “whether FreeRDP appears to be a development version... yes” 后直接 `AC_MSG_ERROR` 退出（exit code 1）。
- 报错信息本身明确提示解决方式为 `rerun configure with the --enable-allow-freerdp-snapshots`。
- 已从上游 `https://raw.githubusercontent.com/apache/guacamole-server/1.6.0/configure.ac` 获取 1.6.0 tag 的 configure.ac 验证：其中 `AC_ARG_ENABLE(allow_freerdp_snapshots, [AS_HELP_STRING([--enable-allow-freerdp-snapshots], ...)])`，确认选项名精确存在且受支持。
- 本次修改仅追加一个 configure 选项，不改变原有 `--disable-guaclog`，`ARG` 多行续行符使用正确，shell 展开后两个选项均可正常传递。

## 潜在风险
无。该选项为 guacamole-server 官方提供的显式放行开关，仅放宽对开发版 FreeRDP 的校验，不改变其他构建行为；不会影响已成功构建的其他 guacd 版本目录（未修改其文件）。
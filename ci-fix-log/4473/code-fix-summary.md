# 修复摘要

## 修复的问题
修复 guacd 1.6.0（24.03-LTS-SP4）镜像构建时 `./configure` 因检测到开发版 FreeRDP 而报错退出的问题。

## 修改的文件
- `Others/guacd/1.6.0/24.03-lts-sp4/Dockerfile`: 在 `GUACAMOLE_SERVER_OPTS` 中追加 `--enable-allow-freerdp-snapshots` 选项。

## 修复逻辑
CI 日志显示 guacamole-server 1.6.0 的 `configure` 检测到 openEuler 24.03-LTS-SP4 仓库提供的 FreeRDP 为开发/快照版本（`checking whether FreeRDP appears to be a development version... yes`），随即主动报错退出，并明确提示需传入 `--enable-allow-freerdp-snapshots` 方可通过。

原第 7-8 行的多行字符串仅设置了 `--disable-guaclog`，未包含该放行开关。修复方式为在该多行变量中追加 `--enable-allow-freerdp-snapshots`，保持既有换行/转义写法不变（使用行尾 `\` 续行），不破坏 Dockerfile 语法，也未改动其他构建逻辑。该参数即 configure 报错信息直接给出的官方放行开关，无需拉取上游源码验证正则。

## 潜在风险
无。改动仅对 SP4 版本 Dockerfile 的 configure 参数生效，SP1/SP2 及 README/meta.yml/image-info.yml 均未受影响；追加选项是 guacamole-server 1.6.0 官方支持的 configure 选项，不会改变其他构建行为。
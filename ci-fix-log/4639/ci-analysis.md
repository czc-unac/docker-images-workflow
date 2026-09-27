# CI 失败分析报告

## 基本信息
- PR: #4639 — 【自动升级】guacd容器镜像升级至1.6.0版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: FreeRDP开发版校验失败
- 新模式症状关键词: `development version of FreeRDP`, `--enable-allow-freerdp-snapshots`, `configure: error`, `guacamole-server`

## 根因分析

### 直接错误
```
#8 10.37 checking for freerdp2 freerdp-client2 winpr2... yes
#8 10.38 checking whether FreeRDPConvertColor is declared... yes
#8 10.43 checking whether FreeRDP appears to be a development version... yes
#8 10.49 configure: error:
#8 10.49   --------------------------------------------
#8 10.49    You are building against a development version of FreeRDP. Non-release
#8 10.49    versions of FreeRDP may have differences in behavior that are impossible to
#8 10.49    check for at build time. This may result in memory leaks or other strange
#8 10.49    behavior.
#8 10.49
#8 10.49    *** PLEASE USE A RELEASED VERSION OF FREERDP IF POSSIBLE ***
#8 10.49
#8 10.49    If you are ABSOLUTELY CERTAIN that building against this version of FreeRDP
#8 10.49    is OK, rerun configure with the --enable-allow-freerdp-snapshots
#8 10.49   --------------------------------------------
#8 ERROR: process "/bin/sh -c cd /tmp && curl ... && ./configure --prefix=\"$PREFIX_DIR\" $GUACAMOLE_SERVER_OPTS && ..." did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Others/guacd/1.6.0/24.03-lts-sp4/Dockerfile:25`（`./configure` 步骤，由第 8 行 `ARG GUACAMOLE_SERVER_OPTS` 控制）
- 失败原因: guacamole-server 1.6.0 的 `configure` 检测到基础镜像中 `freerdp`/`freerdp-devel` 提供的 FreeRDP 是**开发版（development / snapshot）**，默认拒绝继续配置；构建时 `configure` 仅传入 `--disable-guaclog`，未传入 `--enable-allow-freerdp-snapshots`，因此配置阶段报错、exit code 1，后续 `make`/`make check`/`make install` 均未执行。

### 与 PR 变更的关联
- 本 PR 新增文件 `Others/guacd/1.6.0/24.03-lts-sp4/Dockerfile`（`pr.diff` 中 `new_file: True`, 新增 40 行）。
- 该 Dockerfile 第 8 行定义 `ARG GUACAMOLE_SERVER_OPTS="--disable-guaclog"`，第 25 行 `./configure --prefix="$PREFIX_DIR" $GUACAMOLE_SERVER_OPTS`。
- 24.03-lts-sp4 仓库中的 FreeRDP 被 configure 识别为开发版，而该 Dockerfile 未加放行开关，直接触发失败。属于本次新增 Dockerfile 引入的问题，与 PR 改动直接相关。
- README.md / doc/image-info.yml / meta.yml 的条目新增为配套元数据，与构建失败无因果关系。

## 修复方向

### 方向 1（置信度: 高）
在 `Others/guacd/1.6.0/24.03-lts-sp4/Dockerfile` 的 `GUACAMOLE_SERVER_OPTS`（第 8 行）中追加 `--enable-allow-freerdp-snapshots`，使 configure 显式放行开发版 FreeRDP。该选项正是日志中 configure 明确提示的解决方式，且 configure 检测已确认 `freerdp2 freerdp-client2 winpr2... yes`，其余 RDP 相关依赖均可用。

### 方向 2（可选）
若上游/项目规范不接受使用开发版 FreeRDP，则应调整基础镜像中安装的 freerdp 包版本，改用发行版（released）FreeRDP；但该路径依赖 openEuler 仓库实际可提供的包版本，成本更高，故优先采用方向 1。

## 需要进一步确认的点
- 确认其他成功构建的 guacd 版本（如 `1.6.0/24.03-lts-sp2`、`1.5.5/24.03-lts`）Dockerfile 是否已包含 `--enable-allow-freerdp-snapshots`，以保持各版本配置一致。
- README.md 与 image-info.yml 中新增的 `1.6.0-oe2403sp4` 条目声明架构为 `amd64, arm64`；需确认 aarch64 构建是否同样会遇到 FreeRDP 开发版校验（预期一致，同一 configure 逻辑）。
- 日志末尾为 `Finished: FAILURE`，失败 job 为 x86_64 构建，构建错误真实且可复现于当前日志，非 trigger 层成功态误判。

## 修复验证要求
本修复不涉及对第三方/上游源文件正则的 patch，无需额外拉取上游文件验证。但 code-fixer 提交前应确认：
- `--enable-allow-freerdp-snapshots` 是 guacamole-server 1.6.0 `configure`（源码 `configure.ac`）实际支持的选项名（日志中已由 configure 自身提示该确切选项名，可直接采用）。
- 修改后的 `GUACAMOLE_SERVER_OPTS` 在 shell 展开、引号嵌套上不破坏原有 `--disable-guaclog`。

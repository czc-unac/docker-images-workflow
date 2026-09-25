# CI 失败分析报告

## 基本信息
- PR: #4473 — 【自动升级】guacd容器镜像升级至1.6.0版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: FreeRDP快照校验拒绝
- 新模式症状关键词: development version of FreeRDP, --enable-allow-freerdp-snapshots, configure: error, guacamole-server

## 根因分析

### 直接错误
```
#8 10.28 checking for freerdp2 freerdp-client2 winpr2... yes
#8 10.29 checking whether FreeRDPConvertColor is declared... yes
#8 10.34 checking whether FreeRDP appears to be a development version... yes
#8 10.40 configure: error:
#8 10.40   --------------------------------------------
#8 10.40    You are building against a development version of FreeRDP. Non-release
#8 10.40    versions of FreeRDP may have differences in behavior that are impossible to
#8 10.40    check for at build time. This may result in memory leaks or other strange
#8 10.40    behavior.
#8 10.40
#8 10.40    *** PLEASE USE A RELEASED VERSION OF FREERDP IF POSSIBLE ***
#8 10.40
#8 10.40    If you are ABSOLUTELY CERTAIN that building against this version of FreeRDP
#8 10.40    is OK, rerun configure with the --enable-allow-freerdp-snapshots
#8 10.40   --------------------------------------------
#8 ERROR: process "/bin/sh -c cd /tmp && curl -fSL -o guacamole-server.tar.gz ... ./configure --prefix=\"$PREFIX_DIR\" $GUACAMOLE_SERVER_OPTS && make -j $(nproc) && make check && make install" did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Others/guacd/1.6.0/24.03-lts-sp4/Dockerfile`:25（`./configure` 步骤）；对应 `GUACAMOLE_SERVER_OPTS` 定义在第 5-7 行（当前仅含 `--disable-guaclog`）
- 失败原因: openEuler 24.03-LTS-SP4 仓库提供的 FreeRDP 是**开发版/快照版**，guacamole-server 1.6.0 的 `configure` 脚本检测到后主动报错退出，要求显式传入 `--enable-allow-freerdp-snapshots` 才允许继续构建。

### 与 PR 变更的关联
- 本 PR 新增了 `Others/guacd/1.6.0/24.03-lts-sp4/Dockerfile`（从既有 1.6.0 模板复制并适配 sp4 基础镜像），构建时基于 sp4 基础镜像安装的 `freerdp`/`freerdp-devel` 为开发版本。
- 该 Dockerfile 的 `GUACAMOLE_SERVER_OPTS` 仅设置了 `--disable-guaclog`，未包含 `--enable-allow-freerdp-snapshots`，因此 `./configure` 在检测到开发版 FreeRDP 后终止，属于本次新增文件直接触发。
- 同目录的 sp2 版本（`1.6.0/24.03-lts-sp2`）未出现在失败中，推测其基础镜像中的 FreeRDP 为 release 版本；sp4 仓库的包版本变化导致同一构建逻辑在 sp4 上失败。

## 修复方向

### 方向 1（置信度: 高）
在 `Dockerfile` 的 `GUACAMOLE_SERVER_OPTS` 中追加 `--enable-allow-freerdp-snapshots`，使 guacamole-server 1.6.0 允许基于 openEuler 24.03-LTS-SP4 提供的 FreeRDP 快照版构建。该参数正是 configure 报错信息明确给出的官方放行开关。

### 方向 2（可选，置信度: 中）
若不接受快照版 FreeRDP，则需在基础镜像中安装/编译一个 release 版本的 FreeRDP（如指定发行版源码包），替代 openEuler 仓库自带的开发版，使 configure 的版本检测通过。此方向改动量更大，通常不优先在自动升级场景采用。

## 需要进一步确认的点
- 确认 `Others/guacd/1.6.0/24.03-lts-sp2/Dockerfile` 与新增 sp4 版本的差异：若 sp2 已包含 `--enable-allow-freerdp-snapshots`，则应保持各 OS 版本构建参数一致，仅补齐 sp4 缺失项。
- 确认 openEuler 24.03-LTS-SP4 仓库中 `freerdp`/`freerdp-devel` 的实际版本号，判断其确为快照/开发版（日志中 `checking whether FreeRDP appears to be a development version... yes` 已给出结论）。
- 确认仓库规范中 `Others/guacd/meta.yml`、`doc/image-info.yml`、`README.md` 的新增条目（路径、tag 格式）与目录实际路径一致；本次 CI 失败发生在 Docker build 阶段，未涉及 YAML 校验阶段。

## 修复验证要求
本次修复不涉及对第三方/上游源文件正则的 patch 匹配，无需拉取上游源码验证正则。但 code-fixer 提交前应确认：
- `--enable-allow-freerdp-snapshots` 为 guacamole-server 1.6.0 `configure` 脚本实际支持的选项（日志中 configure 报错已直接给出该选项名，可直接采用）。
- 修改后 `GUACAMOLE_SERVER_OPTS` 的换行/转义写法不破坏 Dockerfile 语法（原有 `--disable-guaclog` 位于多行字符串中）。

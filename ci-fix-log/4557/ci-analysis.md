# CI 失败分析报告

## 基本信息
- PR: #4557 — 【自动升级】guacd容器镜像升级至1.6.0版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: FreeRDP开发版被拒
- 新模式症状关键词: development version of FreeRDP, --enable-allow-freerdp-snapshots, configure: error, guacamole-server, make check

## 根因分析

### 直接错误
```
#8 10.87 checking for freerdp2 freerdp-client2 winpr2... yes
#8 10.88 checking whether FreeRDPConvertColor is declared... yes
#8 10.93 checking whether FreeRDP appears to be a development version... yes
#8 10.99 configure: error:
#8 10.99   --------------------------------------------
#8 10.99    You are building against a development version of FreeRDP. Non-release
#8 10.99    versions of FreeRDP may have differences in behavior that are impossible to
#8 10.99    check for at build time. This may result in memory leaks or other strange
#8 10.99    behavior.
#8 10.99
#8 10.99    *** PLEASE USE A RELEASED VERSION OF FREERDP IF POSSIBLE ***
#8 10.99
#8 10.99    If you are ABSOLUTELY CERTAIN that building against this version of FreeRDP
#8 10.99    is OK, rerun configure with the --enable-allow-freerdp-snapshots
#8 10.99   --------------------------------------------
#8 ERROR: process "/bin/sh -c cd /tmp && curl -fSL ... && ./configure --prefix="$PREFIX_DIR" $GUACAMOLE_SERVER_OPTS && make -j $(nproc) && make check && make install" did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Others/guacd/1.6.0/24.03-lts-sp4/Dockerfile:22-26`（新增文件的 `RUN ... ./configure --prefix="$PREFIX_DIR" $GUACAMOLE_SERVER_OPTS ...` 步骤）
- 失败原因: openEuler 24.03-LTS-SP4 基础镜像 `yum install` 安装的 `freerdp/freerdp-devel` 被 guacamole-server 1.6.0 的 configure 脚本识别为 **development/快照版 FreeRDP**，而 `GUACAMOLE_SERVER_OPTS` 仅包含 `--disable-guaclog`，未传入 configure 明确要求的 `--enable-allow-freerdp-snapshots`，因此 configure 直接报错退出。

### 与 PR 变更的关联
直接相关。本 PR 新增 `Others/guacd/1.6.0/24.03-lts-sp4/Dockerfile`，该 Dockerfile 第 3 行将构建选项硬编码为仅 `--disable-guaclog`，第 22-26 行在原 openEuler 24.03-LTS-SP4 基础镜像上编译 guacamole-server 1.6.0 时触发上述 configure 检查。CI 失败由此新增文件触发，不是基础设施问题。

未发现其他失败点：日志中 `perl-Error` 安装、libtool/autoreconf、各 `checking for ...` 均为正常输出；`JSONArgsRecommended`（CMD 建议 JSON 格式）仅为 lint 警告，非致命；日志末尾为 `Build step 'Execute shell' marked build as failure` / `Finished: FAILURE`，为真实构建失败。

## 修复方向

### 方向 1（置信度: 高）
在 guacamole-server 的 configure 阶段显式允许使用 FreeRDP 快照/开发版本，即将 `ARG GUACAMOLE_SERVER_OPTS` 中补充 configure 明确提示的 `--enable-allow-freerdp-snapshots` 选项（保留原有的 `--disable-guaclog`）。日志已逐字给出该必需参数，属于 configure 自身的官方开关，风险最低。

### 方向 2（可选，置信度: 中）
不修改 configure 选项，改为在 `yum install` 中安装 openEuler 24.03-LTS-SP4 仓库内的 **release 版 FreeRDP**（而非被识别为开发版的 freerdp 包）。此方向依赖该基础镜像仓库是否提供符合 guacamole 1.6.0 要求的正式版 freerdp，需先核实包版本；若仓库只有开发版包，则仍应回到方向 1。

## 需要进一步确认的点
- 确认 openEuler 24.03-LTS-SP4 仓库中 `freerdp`/`freerdp-devel` 的实际版本号，判断其是否确为开发/快照版（日志已判定为开发版，建议二次确认）。
- 确认同一 guacd 1.6.0 在 `24.03-lts-sp2` 的既有 Dockerfile 是否采用了不同构建选项（作为同项目参照），以判断本新增文件是否遗漏了该参数。
- 确认 guacamole-server 1.6.0 configure 中 `--enable-allow-freerdp-snapshots` 选项的准确拼写与作用范围（日志已给出，需与上游 1.6.0 源码一致）。

## 修复验证要求
本次修复为 Dockerfile 内构建选项调整，不涉及对第三方/上游源文件的正则 patch，无强制上游拉取验证要求。但 code-fixer 提交前建议确认 `--enable-allow-freerdp-snapshots` 与 guacamole-server 1.6.0 的 configure 选项完全一致；若选择方向 2，需先确认基础镜像仓库中 freerdp 的实际版本。

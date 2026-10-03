# CI 失败分析报告

## 基本信息
- PR: #4852 — 【自动升级】jetty容器镜像升级至12.1.14版本.
- 失败类型: build-error（候选，未证实）
- 置信度: 低
- 知识库匹配: 模式42（日志缺失无法定位）
- 新模式标题: 缺少curl依赖
- 新模式症状关键词: curl: command not found, Cannot open: No such file or directory, wget, curl, jetty-home

> ⚠️ 证据状态说明：本次上下文 `ci.logs` 为 `(not available — analyze based on PR diff only)`，
> 未提供任何失败 job 的日志，也未提供 `ci.run_info`。因此无法引用真实报错行，
> 以下分析仅为基于 `pr.diff` 的静态推断，**根因未证实**。

## 根因分析

### 直接错误
无法给出。`ci.logs` 为空（未提供），无法复制任何真实错误信息。

从 PR diff 可静态观察到的可疑点（非日志证据）：
- 新增 `Others/jetty/12.1.14/24.03-lts-sp4/Dockerfile` 第 6-8 行的安装语句为：
  `dnf install -y wget git java-17-openjdk shadow-utils`
  —— 只安装了 `wget`，**未安装 `curl`**。
- 同一 Dockerfile 第 17 行下载 jetty 发行包时使用的是 `curl -SL ... -o jetty.tar.gz`。
- 该 RUN 块内所有命令之间使用 `;`（而非 `&&`）连接，且未 `set -e`，因此一旦 `curl` 不存在，
  失败会被后置的 `tar`、`sed`、`start.jar` 等命令连锁触发，最终以最后一条
  `java -jar "$JETTY_HOME/start.jar" --list-config` 的非零退出码结束。

### 根因定位
- 失败位置（静态推断）: `Others/jetty/12.1.14/24.03-lts-sp4/Dockerfile:6-8`（安装列表）与 `:17`（curl 调用）之间的不一致
- 失败原因（推断）: 构建镜像中未安装 `curl`，导致 jetty-home 发行包下载失败、后续 `tar`/`start.jar` 连锁失败。

### 与 PR 变更的关联
本 PR 为自动升级，新增了整个 `12.1.14/24.03-lts-sp4/` 目录（Dockerfile 与两个脚本全新文件）。
若基础镜像 `openeuler/openeuler:24.03-lts-sp4` 不自带 `curl`，则新增 Dockerfile 中的
`curl` 调用会直接导致本次构建失败，与 PR 改动强相关。

## 修复方向

### 方向 1（置信度: 中）
在第一个 `dnf install` 步骤中补充 `curl` 包，使下载步骤所需的命令存在。
（仅描述思路，不提供代码。若上游 12.1.13 版本可以构建成功，应对比其安装列表确认是否原本就安装过 curl。）

### 方向 2（置信度: 中）
若不希望改变依赖列表，可将下载命令由 `curl -SL` 改为已安装的 `wget` 等价写法，
并确认变量替换后的 URL 与输出文件名一致。

### 方向 3（置信度: 低）
检查 `12.1.14` 版本在 Maven Central 是否真实存在，以及 `start.jar --create-startd`
的 `--add-to-start` 模块列表（`server,http,ext,resources,deployment-scanner,ee8-deploy,ee8-webapp`）
在 12.1.14 中是否全部有效（模块更名/移除会导致 `--create-startd` 失败）。

### 方向 4（置信度: 低）
新增 Dockerfile 没有任何 Copyright / SPDX 头，而两个 `.sh` 文件使用 Intel 版权头 +
`Eclipse Public License - v 2.0`，与仓库规范（模式17，MulanPSL-2.0）不一致，需确认
`check_package_license` 预检是否对新增 Dockerfile 生效。

## 需要进一步确认的点
1. **首要**：获取本次 CI 真正失败 job 的日志（构建 job，而非 trigger/编排层），确认首个
   `error`/非零退出码出现在哪一步。当前 `ci.logs` 为空，无法定位。
2. 确认 `openeuler/openeuler:24.03-lts-sp4` 基础镜像是否自带 `curl`；如需从构建日志确认，
   可搜索 `curl: command not found` / `Cannot open: No such file or directory` / `tar:` 关键字。
3. 对比 `Others/jetty/12.1.13/24.03-lts-sp4/Dockerfile` 的安装列表，判断 12.1.13 是否安装了 curl
   （若安装了，则本 PR 新增文件遗漏了 curl，为确定性根因）。
4. 确认 jetty `12.1.14` 是否为 Maven Central 上真实存在的版本。
5. 确认 `--add-to-start` 中 `ext`、`ee8-deploy`、`ee8-webapp` 等模块在 12.1.14 中存在。

## 修复验证要求
本报告置信度为「低」，根因未经日志证实。code-fixer 在提交修复前**必须**执行：
1. 通过 CI 构建日志确认失败步骤，禁止仅凭本报告的 curl 推断直接提交；
2. 确认基础镜像内是否存在 `curl`（可通过上游或 CI 日志核实）；
3. 若采用「补装 curl」方向，需与 `12.1.13` 成功版本的 Dockerfile 安装列表逐项比对，避免引入偏差；
4. 若涉及修改 `--add-to-start` 模块名，必须以 jetty `12.1.14` 实际可用模块列表为准进行验证。

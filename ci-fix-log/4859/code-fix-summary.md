# 修复摘要

## 修复的问题
本次 CI 失败为 **infra-error（CI 基础设施/工具链问题）**，与 PR #4859 的代码变更无关，**无需修改任何源代码**。

- 实际失败 job：`aarch64 / check_build`（Jenkins `multiarch/openeuler/aarch64/openeuler-docker-images` build #5069）
- 实际报错（日志末尾）：
  ```
  File ".../eulerpublisher/update/container/app/update.py", line 367, in <module>
      if obj.check_updates():
  ...
  File ".../update.py", line 90, in _check_app_image
      if subprocess.call([...])
  FileNotFoundError: [Errno 2] No such file or directory: 'eulerpublisher'
  Build step 'Execute shell' marked build as failure
  Finished: FAILURE
  ```
- 该失败发生在 Docker 构建与推送**均已成功之后**，属于 CI 编排工具 `eulerpublisher` 的收尾/更新检查阶段在 aarch64 runner 上找不到 `eulerpublisher` 可执行文件。对应知识库「模式39：CI工具依赖缺失」。

## 修改的文件
- 无（未修改 `pr.changed_files` 中的任何文件）

## 修复逻辑

### 1. 如何拿到真实失败日志（原分析报告拿不到日志的原因）
分析报告称 `ci.logs` / `ci.run_info` 均不可用，是因为日志抓取脚本 `scripts/lib/ci_gitcode_api.py` 中：

```python
_JENKINS_URL_RE = re.compile(r'https?://ci\.openeuler\.openatom\.cn/job/[^\s<>"\')\]\|]+')
```

而 PR 评论里的真实链接域名是 `log-ci.openeuler.openatom.cn`（前缀 `log-`），且 Jenkins 作业路径是嵌套结构 `/job/multiarch/job/openeuler/job/aarch64/job/openeuler-docker-images/`，因此正则未匹配到，导致 `ci.logs=(not available)`。

我通过 PR #4859 的 PR 评论（GitCode v5 API）拿到 Bot 结果表：
- `check_package_license`: ⚠️ WARNING
- `check_sca`: ✅ SUCCESS
- `x86_64 / check_build`: ✅ SUCCESS
- `aarch64 / check_build`: ❌ FAILED

再通过 `https://ci.openeuler.openatom.cn/job/multiarch/job/openeuler/job/aarch64/job/openeuler-docker-images/5069/consoleText` 拉取真实控制台日志。

### 2. 真实根因（已用日志证实）
aarch64 build #5069 中：
```
INFO: [Build] finished          <- Docker 构建成功
INFO: [Push] finished           <- 推送成功
...
FileNotFoundError: [Errno 2] No such file or directory: 'eulerpublisher'
Build step 'Execute shell' marked build as failure
```

即 Docker 镜像 `grafana:13.2.3-oe2403sp4-aarch64` 构建并推送成功，失败仅由 CI 工具 `eulerpublisher` 在其 `update.py` 中调用同名可执行文件时 PATH 查找失败引起。这是 runner 环境/工具安装问题，与本 PR 改动的 5 个文件无关，**任何源码改动都无法修复该失败**。

### 3. 对分析报告三个「可疑方向」的逐条证伪
分析报告仅有 diff、无日志，只能推测。经实际验证，三个方向均不成立：

- **方向1（BUILDARCH 冲突）不成立**：本地用 BuildKit 实际构建 13.2.3 镜像并通过 `docker build` 验证，`RUN` 内对 `BUILDARCH` 的赋值在 shell 中生效（`AFTER: BUILDARCH=x86_64`），最终下载 `grafana-enterprise-13.2.3-1.x86_64.rpm`。aarch64 日志也显示运行时实际下载的是 `grafana-enterprise-13.2.3-1.aarch64.rpm`（而非 echo 命令文本里的 `arm64`），x86_64/aarch64 两架构 RPM 经 `curl -I` 与本地构建均确认存在、可安装。
- **方向2（续行符尾随空格）不成立**：`docker build --check` 报告 "no warnings found"，且完整 `docker build` + 容器启动均成功；该写法在 `13.2.2` 版本中同样存在且已入库。
- **方向3（缺少 SPDX/Copyright 头）不成立**：本仓库所有 `Cloud/grafana/**/Dockerfile`（含历史成功构建版本）均无 Copyright/SPDX 头；`check_package_license` 本次仅为 WARNING（且提示的是"缺少项目级 Copyright 声明文件"，属仓库级既有告警），并非导致 job FAILED 的检查项。

### 4. 附带发现（非本次 CI 失败原因，故未修改）
aarch64 与 x86_64 日志中均出现 Grafana RPM 的 `%post` scriptlet 报 `find: command not found`（exit status 127），原因是基础镜像 `openeuler/openeuler:24.03-lts-sp4` 默认不含 `findutils`。但 dnf 仍以退出码 0 完成安装（日志随后为 `Complete!` / `DONE`），x86_64 在同样告警下 `check_build` 为 SUCCESS，因此它不是本次失败根因。出于"最小化、只改与 CI 失败直接相关的代码"的约束，本次不做改动。

## 潜在风险
- 无源代码改动，不引入任何新风险。
- 提示流程侧：由于真实失败为 `eulerpublisher` 工具缺失（infra），即使重新触发 CI 仍会在 aarch64 runner 收尾阶段失败；建议由 CI/基础设施团队在 aarch64 构建节点修复 `eulerpublisher` 可执行文件的 PATH/安装，或修复日志抓取脚本对 `log-ci.openeuler.openatom.cn` 及嵌套 `/job/...` 路径的匹配，二者均不属于本仓库源码范畴。
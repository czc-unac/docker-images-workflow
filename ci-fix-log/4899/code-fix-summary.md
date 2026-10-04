# 修复摘要

## 修复的问题
经核查真实 CI 日志，本 PR 失败为 infra-error（x86_64 构建 job 拉取 CI 构建脚本时上游返回 HTTP 429 限流），与代码变更无关，**无需修改任何代码**。

## 修改的文件
- 无（未修改 `pr.changed_files` 中的任何文件）

## 修复逻辑

分析报告因缺少 CI 日志，基于 diff 推断为 `lint-error`（模式17：新增 Dockerfile 缺少 Copyright/SPDX 头）。本次未直接套用该推断，而是通过 GitCode API（PR #4899 评论）与 Jenkins 拉取了真实失败 job 日志进行复核，结论与推断不符：

1. **真实失败位置**：下游 x86_64 构建 job `multiarch/openeuler/x86-64/openeuler-docker-images #5018`（触发 job `#5247`）。其 `consoleText` 全文为：
   ```
   [****-docker-images] $ /bin/bash /tmp/jenkins17808699145680555384.sh
   curl: (22) The requested URL returned error: 429
   chmod: cannot access 'build.sh': No such file or directory
   /tmp/jenkins17808699145680555384.sh: line 21: ./build.sh: No such file or directory
   Build step 'Execute shell' marked build as failure
   Finished: FAILURE
   ```
   CI 编排脚本在 `./build.sh` 之前拉取构建脚本 `build.sh` 时被上游以 **HTTP 429（Too Many Requests，限流）** 拒绝，导致 `build.sh` 缺失、脚本无法执行。该步骤发生在任何 Docker 构建之前，与 `Others/npm/12.2.0/24.03-lts-sp4/Dockerfile` 内容无任何关系。

2. **同代码 aarch64 构建成功**：同一 PR 的 `multiarch/openeuler/aarch64/openeuler-docker-images #5114` 结果为 `Finished: SUCCESS`，日志显示完整构建成功：
   - `curl ... node-v22.23.2-linux-arm64.tar.xz` 下载成功（28.8M）
   - `RUN npm install -g npm@12.2.0` 成功（`changed 92 packages in 11s`）
   - 镜像 `openeulertest/npm:12.2.0-oe2403sp4-aarch64` 构建并推送成功
   同一份代码在 aarch64 上通过，证明 `Dockerfile`/元数据本身正确，x86_64 失败属环境性偶发。

3. **`check_package_license` 仅为 WARNING**：触发 job `#5247` 的 `check result` 为
   `[{"name": "check_package_license", "result": 1, "details": ["仓库copyright检查未通过: 缺少项目级Copyright声明文件"]}, {"name": "check_sca", "result": 0}]`。
   该告警是**仓库项目级**缺少 Copyright 声明文件（master 上同样存在），并非本 PR 新增文件缺头，且为 WARNING（触发 job 最终 `Finished: SUCCESS`）。因此模式17 不构成本次失败根因；为新增 Dockerfile 单独补 SPDX 头既不能修复该告警，也与 npm 家族既有 Dockerfile（如 12.1.0、12.0.2 均无头）不一致，属无关改动。

4. **上游版本核实**（排除模式02/19“版本不存在”）：
   - `registry.npmjs.org/npm/12.2.0` 存在（`version: 12.2.0`，`engines.node: ^22.22.2 || ^24.15.0 || >=26.0.0`）
   - `nodejs.org/dist/v22.23.2/node-v22.23.2-linux-{x64,arm64}.tar.xz` 均返回 HTTP 200
   - 即 `NODE_VERSION=22.23.2` 与 `VERSION=12.2.0` 组合合法，Dockerfile 逻辑无误。

综合以上，本失败对应 infra-error（CI 构建脚本拉取被限流 429），参照知识库模式39「CI 工具依赖缺失 / infra-error 与代码无关」的处理原则，不修改代码。

## 潜在风险
无。未做任何代码改动，不引入新风险。建议对 x86_64 构建 job 重跑（re-trigger）以确认 429 为偶发限流；若重跑仍为 429，应由 CI/基础设施侧排查 `build.sh` 下载源的限流策略。
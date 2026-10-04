# 修复摘要

## 修复的问题
无需代码修改：CI 分析报告判定为 `infra-error`（证据不足、无失败日志），且对报告提出的三个疑点逐一核验后均不成立。

## 修改的文件
- 无

## 修复逻辑
CI 分析报告明确指出：`ci.logs` 为 "(not available — analyze based on PR diff only)"，`ci.run_info` 亦不可用，失败类型为 `infra-error`，置信度为"低"，并明确要求"在拿到失败 job 真实日志前不建议 Code Fixer 盲改"。因此本次不实施任何修改。

为排除报告中的三个待验证疑点，本次已做如下实证核验：

1. **疑点A（许可证头缺失）— 不成立**：对比既有同目录同样式文件 `Others/npm/12.1.0/24.03-lts-sp4/Dockerfile`、`Others/npm/12.0.2/24.03-lts-sp4/Dockerfile`，它们的结构与本 PR 新增的 Dockerfile 完全一致（均以 `ARG BASE=openeuler/openeuler:24.03-lts-sp4` 开头，均无 Copyright/SPDX 头）。既有文件已通过 CI，说明该路径并不要求许可证头，新模式并无回归。

2. **疑点B（npm@12.2.0 不存在）— 不成立**：实测 `https://registry.npmjs.org/npm/12.2.0` 返回有效的版本元数据（含 `dist.tarball: https://registry.npmjs.org/npm/-/npm-12.2.0.tgz`），说明上游 npm 12.2.0 存在，`npm install -g npm@12.2.0` 不会因版本不存在而失败。

3. **疑点C（Node v22.23.2 不存在）— 不成立**：实测 `https://nodejs.org/dist/v22.23.2/node-v22.23.2-linux-x64.tar.xz` 返回 HTTP 200（content-length 31058332），说明该 Node 版本 tarball 存在，下载阶段不会 404。

另外，`NODEARCH` 由 `TARGETARCH` 经 `sed 's/amd64/x64/'` 派生，未使用与 BuildKit 冲突的 `BUILDARCH`，不存在模式09问题；`meta.yml` 与 `doc/image-info.yml` 的新增条目格式与既有条目一致。

综上，PR 变更内容与既有文件模式一致，未发现可归因于代码的失败根因。当前失败信息不足，强行修改会引入无依据的改动，故不做任何代码修改，建议补充失败 job 的完整日志（x86-64/aarch64 构建 job 或预检 stage 日志）后再行定位。

## 潜在风险
无。本次未修改任何源码文件，不影响现有功能与构建流程。
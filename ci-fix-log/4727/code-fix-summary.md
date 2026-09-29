# 修复摘要

## 修复的问题
无需代码修改：CI 失败分析报告判定为 `infra-error`（证据不足，`ci.logs` 与 `ci.run_info` 均不可用，无法定位真实失败点），与本次 PR 的代码变更无确定关联。

## 修改的文件
（无）

## 修复逻辑
分析报告明确标注 `失败类型: infra-error`、`置信度: 低`，并说明 `ci.logs` 为 `(not available — analyze based on PR diff only)`、`ci.run_info` 为 `(not available)`，即不存在任何可引用的错误信息。报告中的 4 个候选可疑点均为“基于 diff 的候选推断”，报告本身也在结论中要求“在未获得真实 CI 日志前，不应基于候选推断直接修改 Dockerfile”。

按照项目设计（system-design.md 场景 D “证据不足（降级路径）”、data-model.md 4.2/5.2），当失败类型为 `infra-error` 时，code-fix 阶段不做代码修改，仅输出说明摘要并设置 `no_changes=true`，不创建 Fix PR。

为排除报告中“最确定的差异点”，已对候选可疑点逐条核实，均不成立：
- 候选 1（Copyright/SPDX 缺失）：仓库内 2046 个 Dockerfile 中仅 126 个含 SPDX 头，同路径的 `2025.2/24.03-lts-sp4/Dockerfile` 同样无版权头且已成功合入，说明 Dockerfile 并不强制要求版权头。
- 候选 2（`support/v2026.2` 分支不存在）：已通过 `git ls-remote` 核实上游 `cp2k/cp2k` 同时存在 `refs/heads/support/v2026.2` 与 `refs/tags/v2026.2`，clone 目标有效。
- 候选 3（缺少 python3）：`2026.2` 的 `install_cp2k_toolchain.sh` 未引用 python，且同基础镜像的 `2025.2` 版本同样未安装 python3 却能构建，非差异点。
- 候选 4（格式性/`unlink` 返回码）：报告已判定非致命。

因此，在没有真实构建日志（尤其下游 x86-64 / aarch64 架构构建 job 日志）之前，不做任何无依据的修改，避免引入新问题。本次未修改任何文件。

## 潜在风险
无。本次未产生任何代码变更。
# 修复摘要

## 修复的问题
CI 失败分析报告判定为 `infra-error`（证据不足），未提供任何 CI 日志与运行信息，无法定位根因，因此本轮不进行代码修改。

## 修改的文件
- 无

## 修复逻辑
分析报告明确说明：
- `ci.logs` 与 `ci.run_info` 均不可用，无法截取首个错误行，也无法确认失败发生在哪个 job / 阶段；
- 失败类型判定为 `infra-error`，置信度低，知识库匹配“模式42（日志缺失无法定位）”；
- 修复方向明确指出“在获取日志前不建议进行任何代码修改”，并提示“若后续确认日志末尾出现 `Finished: SUCCESS` / `Build successful` 而 PR 仍为失败态，则按 infra-error 处理，Code Fixer 无需改动 Dockerfile 或元数据”。

按照“如果分析报告指出是 infra-error，在 output_file 中说明无需代码修改，不要强行改代码”的约束，本轮不做任何改动。

补充核查（仅确认现状，未修改）：当前 `fix/4730` 分支上 `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile` 已包含上一轮的既有修复——移除了 `gcc-toolset-14-c++*`（该 glob 无对应包；`gcc-toolset-14-gcc*` 已覆盖 gcc-c++），属于合理的最小化改动。元数据侧 `meta.yml` 新增 `1.30.0-oe2403sp4` 条目、`README.md` / `doc/image-info.yml` 版本行格式均与既有条目一致，路径 `1.30.0/24.03-lts-sp4/Dockerfile` 实际存在，`AI/image-list.yml` 中 `onnxruntime: onnxruntime` 条目存在，未发现路径/格式一致性错误。

## 潜在风险
无。本轮未做任何代码改动，不会引入新风险。建议后续补全 x86-64 与 aarch64 架构 job 的构建日志后再判断是否需要进一步修复。
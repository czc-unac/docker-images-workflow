# 修复摘要

## 修复的问题
无需代码修改。CI 分析报告证据不足（无任何失败 job 日志），失败类型倾向 `infra-error`，在获取真实日志前不应执行代码修改。

## 修改的文件
- 无（本轮未对任何文件实施新的修改）

## 修复逻辑
CI 失败分析报告 (`ci_analysis`) 明确给出：
- `ci.logs` 与 `ci.run_info` 均为空，无法定位失败发生的阶段（构建 / 测试 / 预检）；
- 失败类型判定为 `infra-error`（置信度低），根因无法确认；
- 报告结论为"证据不足……在获取失败 job 的真实日志前，不应执行任何代码修改"。

依据 Code Fixer 的核心约束"如果分析报告指出是 infra-error，说明无需代码修改，不要强行改代码"，本轮不做任何代码改动，避免按低置信度假设（版权头、上游 tag、缺失 -devel 依赖）盲目修改引入新问题。

对报告中可独立验证的假设做了外部核验：
- 方向 2（上游 tag 可用性）：已通过 `git ls-remote --tags https://github.com/ceph/ceph.git` 验证 `refs/tags/v21.3.0` 确实存在（commit `b44498fd6d18e36065848c80e00edb64acae1adf`），故 Dockerfile 中 `git clone -b v21.3.0 ... ceph.git` 的 tag 假设不成立，无需修改。
- 方向 1（版权头）与方向 3（缺失 -devel 依赖）：均无法在缺少失败日志的情况下证实，未做改动。

说明：当前 fix 分支工作区已包含上游历史流程提交的既有改动（为 21.3.0 Dockerfile 补充 `protobuf-devel protobuf-compiler grpc-devel grpc-plugins jq`），本轮未新增、未回退该改动，保持最小干预。

## 潜在风险
无。本轮未修改任何代码，不影响现有功能。若后续补充到真实失败日志并能定位到构建阶段错误，应基于日志再行修复。
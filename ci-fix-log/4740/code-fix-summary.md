# 修复摘要

## 修复的问题
本次未做任何代码修改。CI 失败分析报告将其归类为 `infra-error`（置信度低、日志缺失），报告明确要求"在获得真实 CI 日志之前，禁止 code-fixer 依据本报告对 Dockerfile 或 entrypoint.sh 做任何修改"。

## 修改的文件
- 无。
- 已检查但未改动（`pr.changed_files` 允许范围）：
  - `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`
  - `Storage/ceph/21.3.0/24.03-lts-sp4/entrypoint.sh`
  - `Storage/ceph/README.md`
  - `Storage/ceph/doc/image-info.yml`
  - `Storage/ceph/meta.yml`

## 修复逻辑
1. 上下文中的 `ci.logs` 完全缺失（`ci.run_info`、`ci.logs` 均为 `(not available)`），无法定位失败阶段（预检 / Build / Push / Check）与具体 job（x86-64 / aarch64），因此不存在可据以修复的确定性根因。
2. 报告本身已给出处理结论：方向 2 —— 按 `infra-error`（CI 基础设施/日志采集问题）处理，Code Fixer 无需改动代码。
3. 为排除报告"需要进一步确认点 3"中提出的候选风险，已在上游验证 `git ls-remote --tags https://github.com/ceph/ceph.git`，确认 tag `v21.3.0`（commit `b44498fd6d18e36065848c80e00edb64acae1adf`）**真实存在**。故 Dockerfile 中 `git clone -b v${VERSION}`（VERSION=21.3.0）不存在因 tag 缺失导致的 clone 失败，该候选根因被排除，无需修改版本号。
4. 对照仓库既有的 `Storage/ceph/20.3.0/24.03-lts-sp4/{Dockerfile,entrypoint.sh}` 确认：新增 21.3.0 版本在结构与脚本上与已验证可用的 20.3.0 版本保持一致（仅版本号与部分构建开关不同），未发现路径层级、元数据格式或登记条目的明显错误。当前 `fix/4740` 分支还包含此前几轮修复提交（限制 `ninja -j2` 防 OOM、关闭 mgr dashboard 前端、关闭 jaeger tracing），均属于对沙箱环境受限的适配，不应在本轮无日志情况下继续叠加改动。
5. 依据"最小化原则"与"不扩展范围"，在缺少真实失败日志时强行修改 Dockerfile/entrypoint 属于盲目修改，可能引入新问题，故不做代码变更。

## 潜在风险
无代码变更，故无新增风险。后续若要继续定位，需要补齐失败 job 的完整 CI 日志，再判定是构建依赖、超时/资源（OOM/网络）还是预检格式问题，届时再实施最小化修复。
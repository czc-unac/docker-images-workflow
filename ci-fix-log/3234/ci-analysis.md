# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error
- 置信度: 中
- 知识库匹配: 模式11
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
2026-09-15 09:16:26,553 ... eulerpublisher/update/container/app/update.py[line:356]-INFO: Difference: [
    "README.md"
]
2026-09-15 09:16:31,262 ... eulerpublisher/update/container/app/update.py[line:273]-ERROR: There are some specification errors for releasing on appstore in this PR, please check as above.
+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范预检），校验对象为 `README.md`
- 失败原因: 本次 PR 的变更文件列表为 `["README.md"]`，CI 的 appstore 发布规范预检对该路径判定为 `[Path Error]`，期望路径为 `/README.md`，从而报出 "specification errors for releasing on appstore"，导致 job 失败。

### 与 PR 变更的关联
PR #3234 为纯文档改动，仅修改根目录 `README.md`（新增自动新增镜像 Issue 指南，新增 12 行）。日志中 `Difference: ["README.md"]` 与失败项 `README.md` 完全一致，说明失败由本次改动直接触发——CI 的 appstore 发布规范预检对根目录 README.md 的路径校验不通过。

需要说明的是：该检查报告的"期望路径"本身就是 `/README.md`，而实际文件也位于仓库根目录，二者字面一致却仍判 FAILURE，说明校验逻辑可能存在路径归一化/白名单问题（疑似 CI 工具对纯文档 PR 的路径误判）。因此本报告的根因归属存在两种可能，见下方修复方向。

## 修复方向

### 方向 1（置信度: 中）
确认该 CI 规范预检是否允许修改仓库根目录 README.md。若允许，则本次失败属于 CI 工具路径校验误报（infra-error 性质）：
- 代码侧无需修改 PR 内容；
- 需由 CI/工具维护方修复 `update.py` 中 appstore 路径校验逻辑（期望路径判定与实际路径一致却判失败的逻辑缺陷），或为重跑/豁免该检查。

### 方向 2（置信度: 低）
若该 CI 规范预检确实不允许在本仓库根目录 `README.md` 上新增此类非镜像内容（即文档需放置在其他受支持位置），则应调整文档的落点，使其符合 appstore 发布规范允许的路径。

## 需要进一步确认的点
- 查阅 `eulerpublisher/update/container/app/update.py`（尤其 `line:273`、`line:356` 及路径校验相关函数），确认 appstore 规范检查允许的路径白名单，以及为何"期望路径 `/README.md`"与实际路径一致却判 FAILURE。
- 对比其他仅修改根目录 `README.md` 的纯文档 PR 是否同样触发该 `README.md [Path Error]` 失败，以判定是否为已知/普遍的工具误报。
- 获取本次 CI 的完整日志（当前日志为节选），确认预检阶段是否还有其他被截断的校验项。
- 确认该 job 失败是否由 trigger/编排层预检产生，以及是否存在下游真正构建 job（本次日志末尾为 `Finished: FAILURE`，为真实失败）。

## 修复验证要求
本次不涉及"修改正则匹配外部源文件"，无需上游文件拉取验证。但若走方向 2（调整文档落点），code-fixer 必须先从 CI 工具 `eulerpublisher` 的 appstore 路径校验逻辑中确认允许路径规则后再提交，不得假定根目录 README.md 一定不被允许。

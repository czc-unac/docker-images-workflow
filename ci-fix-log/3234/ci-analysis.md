# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（CI appstore 发布规范路径校验失败）
- 置信度: 中
- 知识库匹配: 模式11（YAML / 元数据文件错误，含 appstore 路径校验类失败）
- 新模式标题: (无，归入模式11)
- 新模式症状关键词: (无)

## 根因分析

### 直接错误
```
2026-09-15 09:16:26,553-.../update/container/app/update.py[line:356]-INFO: Difference: [
    "README.md"
]
...
2026-09-15 09:16:31,262-.../update/container/app/update.py[line:273]-ERROR: There are some specification errors for releasing on appstore in this PR, please check as above.
+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范校验阶段），该校验针对变更集 `Difference: ["README.md"]` 逐项核对
- 失败原因: 本次 PR 的唯一变更文件为仓库根目录的 `README.md`，CI 的 appstore 发布规范检查（`ci/container/check`）将其判定为 `[Path Error] The expected path should be /README.md`，规范校验未通过，导致构建被标记为失败。

### 与 PR 变更的关联
- 直接相关。日志中 `Difference` 列表仅含 `README.md`，与 `pr.diff` 中唯一改动文件 `README.md`（新增 12 行"新增应用镜像可通过 Issue 自动化流水线完成"说明）完全一致。校验错误项也明确指向 `README.md`。
- 因此可以确认：失败由本次对根目录 `README.md` 的修改触发（尽管从常规语义看该改动本身仅是文档说明，不含镜像内容）。

## 修复方向

### 方向 1（置信度: 中）
校验工具对根目录 `README.md` 的路径判定为不符合规范（期望路径 `/README.md`）。需先确认该校验对根目录文档变更的准入规则：
- 若该校验仅应针对镜像目录内的条目运行，则应避免让根目录 `README.md` 变更进入 appstore 规范校验范围（属 CI 校验逻辑与文档类变更的兼容问题）。
- 若规范要求修改根 README 时须满足特定的路径/文件约束，则按规范调整 `README.md` 的存放位置或提交方式。

### 方向 2（置信度: 低）
不排除是校验工具对纯文档（README）改动的误报/边界缺陷——同属历史模式11 中 `.claude/agents/README.md` 等 README 路径校验失败的同类现象。若确认为工具缺陷，应标记为与代码无关的规范校验问题。

## 需要进一步确认的点
- 需查阅 `eulerpublisher/update/container/app/update.py` 第 273 行附近及 `ci/container/check` 相关逻辑，确认 `[Path Error] The expected path should be /README.md` 的判定规则：是针对所有变更文件、还是仅针对镜像目录条目。
- 需确认根目录 `README.md` 是否被允许进入 appstore 发布规范校验；若规范本身不期望校验根 README，需要确认这是否为 CI 误报。
- 需确认历史同类 README 路径校验失败（模式11 中 `.claude/agents/README.md` 案例）当时是如何消除的（调整路径还是调整校验范围），以判断本次应走哪条修复路径。
- 日志未显示该 Path Error 之外的其他错误，需确认是否存在被截断的后续校验项。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
不适用（本次失败不涉及对上游第三方源文件的正则/补丁修改）。

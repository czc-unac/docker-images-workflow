# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（appstore 发布规范/路径静态校验）
- 置信度: 中
- 知识库匹配: 模式11（YAML / 元数据文件错误——含 CI appstore 路径校验失败案例）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
2026-09-15 09:16:26,553-.../eulerpublisher/update/container/app/update.py[line:356]-INFO: Difference: [
    "README.md"
]
...
2026-09-15 09:16:31,262-.../eulerpublisher/update/container/app/update.py[line:273]-ERROR: There are some specification errors for releasing on appstore in this PR, please check as above.
+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范预检逻辑；差异清单在 `update.py:356` 输出）
- 失败原因: PR 的变更文件清单（`Difference`）中只有根目录 `README.md`。CI 的 appstore 发布规范校验器对该 README 做路径校验时判定为 `[Path Error] The expected path should be /README.md`，即以 `/README.md` 作为期望路径，而实际传入/识别的路径 `README.md` 与之不一致，校验因此 FAILURE 并阻断流水线。

### 与 PR 变更的关联
- 本 PR 为**纯文档变更**：仅在根目录 `README.md` 中新增“通过 Issue 触发自动化新增镜像”的说明段落（+12 行，未删除、未重命名、非新增文件）。
- 日志中 `Difference` 与失败项均为 `README.md`，与 `pr.diff` 的唯一改动文件完全一致，可确认失败由本 PR 对 `README.md` 的修改触发。
- 但 `README.md` 本就位于仓库根目录，`update.py` 却报期望路径为 `/README.md`，说明校验器对“根目录文档类文件”的路径归一化处理与仓库实际路径表达不一致（缺少前导 `/`），属于校验器侧对根目录 README 的路径比对逻辑问题，而非文档内容本身错误。

## 修复方向

### 方向 1（置信度: 中）
消除触发 appstore 路径校验的根目录 README 变更：确认该校验器对根目录 `README.md` 的期望格式（日志显示期望 `/README.md`），使被校验文件的路径表达与期望一致；若校验器不接受任何根目录文档改动，则应将新增说明迁移到校验器认可的路径/文件中，避免以 `README.md` 作为变更项进入 appstore 预检。

### 方向 2（置信度: 中）
若确认根目录 `README.md` 的修改在流程上合法，则本失败可能来自 CI 校验器（`update.py`）对根路径文件缺少前导 `/` 的归一化缺陷，属基础设施/工具侧问题。此时修复不应改动 PR 文档内容，应反馈 CI 维护方修正预检逻辑（`update.py:273` 所在的路径校验）。

## 需要进一步确认的点
1. `update.py:273` 处路径校验函数的完整实现与期望路径生成规则，确认 `/README.md` 是“带前导斜杠的仓库绝对路径”约定，还是特指某个 appstore README 存放位置的约定。
2. `update.py:356` 输出的 `Difference` 为何将根目录 `README.md` 纳入 appstore 发布预检范围（是否所有 PR 变更文件都会被纳入，或仅特定目录）。
3. 历史上同类“根目录文档改动触发 appstore 路径校验”是否已有既定处理方式（模式11 中 `.claude/README.md` 案例为文件位置不符，而本例文件已在根目录，需区分）。
4. 无法从当前日志确定该校验是“设计上禁止根目录 README 变更”还是“路径归一化 bug”，因此不能 100% 断定应改 PR 还是改 CI。

## 修复验证要求
- 本失败不涉及“修改正则匹配第三方/上游源文件”，无需向上游拉取文件验证正则。
- 因置信度为“中”，code-fixer 在动手前必须先确认上述第 1、2 点（校验器期望路径规则与变更文件纳入范围），再决定修改 PR 文档位置/格式还是走 CI 修复，不能直接假设“移动/重写 README”一定正确。

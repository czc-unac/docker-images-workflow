# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（CI 应用镜像上架规范 / 路径校验）
- 置信度: 中
- 知识库匹配: 模式11（YAML / 元数据文件错误，含 README 路径校验子类）

## 根因分析

### 直接错误
```
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
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 上架规范预检），被校验对象为 `README.md`
- 失败原因: CI 的 appstore 发布规范预检在本次 PR 变更文件列表中命中 `README.md`，并判定其路径不符合期望（`The expected path should be /README.md`），预检汇总出规范错误后直接标记构建失败。

### 与 PR 变更的关联
本次 PR 为**纯文档改动**，`pr.diff` 仅修改仓库根目录的 `README.md`（新增 12 行“通过 Issue 自动新增镜像”的说明，未涉及任何 Dockerfile / meta.yml / image-info.yml / image-list.yml）。CI 日志中 `update.py:356` 打印的 `Difference` 也仅为 `["README.md"]`，与 diff 完全一致，说明失败被直接归因到这一个文档文件。因此该失败**由本 PR 修改根目录 README.md 触发**，但触发它的检查逻辑（appstore 发布规范路径校验）对纯文档 PR 而言可能是误判。

### 影响范围
- 局部：仅 appstore 上架规范预检阶段失败，未进入任何镜像构建/测试阶段（日志无 docker build、无架构 job 报错）。
- 非系统性：不涉及代码、构建依赖或运行环境。

## 修复方向

### 方向 1（置信度: 中）
判定该失败属于 CI 规范预检对“仓库根目录文档类变更”的误触发。需要确认 appstore 上架规范预检的适用边界：它是否只应针对镜像目录（`{image-version}/{os-version}/Dockerfile` 及其元数据）生效，而不应对仓库根 `README.md` 生效。若确属门禁规则问题，应由流水线维护方调整预检范围，而非修改内容。

### 方向 2（置信度: 中）
若预检确实要求 `README.md` 必须位于 `/README.md`（即仓库根）而当前被判定为不匹配，则需核对该 PR 中 README.md 的实际登记路径与 CI 计算出的路径之间的差异来源（例如 diff 路径相对/绝对前缀、大小写、仓库根识别问题），确认为工具路径计算缺陷后再决定由哪一侧修正。

## 需要进一步确认的点
1. `eulerpublisher/update/container/app/update.py:273` 附近 `_check_specification`（appstore 规范校验）的实际判定逻辑：期望路径 `/README.md` 是如何生成的，被校验的实际路径是什么。
2. 该预检的触发条件：是“PR 只要包含 README.md 就校验”，还是“仅在存在待发布镜像条目时才校验”。这决定了本 PR（无任何镜像条目）是否本就不该触发该校验。
3. 是否存在策略：根目录 `README.md` 变更应被识别为纯文档改动并跳过 appstore 规范预检。
4. `update.py:356` 的 `Difference` 打印逻辑与校验路径的一致性问题（为何 diff 为 `README.md` 而期望为 `/README.md`）。
5. 该失败是否在其它纯文档 PR 上可复现，以判断是本次改动特异还是门禁普遍行为。

## 修复验证要求
本次失败为 CI 规范/路径预检，不涉及对第三方/上游源文件的正则 patch，无需执行上游比对验证。若 code-fixer 判定需修改预检范围或路径处理逻辑，必须先复现确认 `README.md` 在预检中的真实路径来源，再验证修复后纯文档 PR 不再触发该路径错误。

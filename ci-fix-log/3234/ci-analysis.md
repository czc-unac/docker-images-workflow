# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: `lint-error`
- 置信度: 中
- 知识库匹配: 模式11（YAML / 元数据文件错误——appstore 路径校验类）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
2026-09-15 09:16:26,553-.../eulerpublisher/update/container/app/update.py[line:356]-INFO: Difference: [
    "README.md"
]
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
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范校验），差异检测位于 `update.py:356`
- 失败原因: 本 PR 的变更集为 `["README.md"]`，CI 的 appstore 发布规范预检对该 README.md 报 `[Path Error] The expected path should be /README.md`，判定路径不符合规范，导致整个构建失败。与知识库模式11中 `.claude/agents/README.md` / `.claude/README.md` 的 appstore 路径校验失败同源。

### 与 PR 变更的关联
- 本 PR 为**纯文档变更**，仅修改仓库根目录 `README.md`（新增「通过 Issue 自动化新增应用镜像」指南），新增 12 行、删除 0 行。
- appstore 发布规范校验把变更文件 `README.md` 当作待发布产物进行路径合规检查，从而报错。即失败是本 PR 触碰 `README.md` 这一文件路径直接触发，而非由文档内容或 Dockerfile/构建逻辑引起。
- 日志中 `Difference: ["README.md"]` 与检查表 `README.md` 一一对应，可确认失败点是该 README 文件本身。

## 修复方向

### 方向 1（置信度: 中）
调整该文档的存放位置/形态，使其不落入 appstore 发布规范校验所禁止的路径。由于该校验似乎要求变更文件位于合法镜像目录结构内，而本 PR 只改动仓库根 `README.md`，可考虑将新增的「new-image 请求指南」内容放到校验允许的位置（例如与镜像文档同级的约定目录），或将该文档拆分为校验白名单内的路径，避免以根 `README.md` 作为唯一变更文件提交。

### 方向 2（置信度: 低）
若确认该校验对「纯文档 PR」存在误判（即根 `README.md` 本应被允许），则属于 CI 编排/校验工具问题，建议由 CI 维护方将该类文档路径加入白名单，Code Fixer 无需改动 Dockerfile 或构建逻辑。

## 需要进一步确认的点
1. `eulerpublisher/update/container/app/update.py` 中 appstore 规范校验的路径白名单/合法路径规则，确认 `README.md` 被判定为 `[Path Error]` 的确切条件（是要求位于镜像目录内，还是要求带前导 `/`）。
2. 判断该校验是否对所有 PR 生效：纯文档 PR 修改根 `README.md` 是否必然失败，还是仅在同时存在镜像变更时才触发。
3. 历史上 `.claude/README.md`（放置于 `.claude/` 根层级）被要求满足的「期望路径」规则，与本例期望 `/README.md` 的规则是否一致。
4. 该 PR 是否本应作为「镜像发布 PR」被校验；若不是，需确认 CI 的 diff 检测为何将根 README 纳入发布校验范围。

> 证据说明：日志仅暴露到校验失败的表层信息，未能给出该校验对 `README.md` 期望路径为 `/README.md` 的完整判定逻辑，故置信度标注为「中」，方向 1/2 均需 code-fixer 先核实校验规则后再提交。

## 修复验证要求
本修复方向涉及对 CI 路径校验规则的理解，但不涉及正则 patch 外部源文件，故无强制上游文件验证要求。
若 Code Fixer 选择修改文档存放路径，需在提交前确认新路径能通过 appstore 规范校验（即以 CI 同款校验逻辑复核路径合法）；若判定为校验工具误判，则不应改动本仓库文件。

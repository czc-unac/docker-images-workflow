# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（CI 规范/路径静态校验失败）
- 置信度: 中
- 知识库匹配: 模式11（YAML / 元数据文件错误——含 README 路径规范校验子类）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

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
日志前半段还显示 CI 本次识别的改动集合：
```
2026-09-15 09:16:26,553 .../update.py[line:356]-INFO: Difference: [
    "README.md"
]
```

### 根因定位
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范校验入口），错误内容由 path 校验逻辑产出（同类历史定位见模式29 `eulerpublisher/update/container/app/format.py:101`）。
- 失败原因: PR 仅改动仓库根目录的 `README.md`，被 CI 的 appstore 发布规范预检当作待发布规格文件处理，并报出 `README.md` 路径不符（期望 `/README.md`），从而判定 `FAILURE`。日志中除这一条 README 路径错误外，没有任何构建/编译/依赖/测试类错误。

### 与 PR 变更的关联
PR 为纯文档变更（`README.md`，新增 12 行、删除 0 行），未新增/修改任何镜像目录（Dockerfile、meta.yml、image-info.yml 等）。`Difference` 列表也仅含 `README.md`，与 `pr.diff` 完全一致，说明失败由本次 README.md 改动直接触发，属于 CI 校验逻辑与"根目录 README 文档变更"之间的判定冲突，而非代码/构建缺陷。

## 修复方向

### 方向 1（置信度: 中）
该失败为 CI appstore 发布规范预检对"根目录 README.md 文档变更"产生的路径判定问题，与镜像发布无关。建议由 Code Fixer / 维护者确认该预检对纯文档 PR 的处理策略，使 docs-only 的根目录 README 变更不被纳入 appstore 发布规格校验（或直接判定为 CI 误报，不修改仓库文件）。

### 方向 2（置信度: 低）
若确认该校验强制要求 `README.md` 必须位于某个具体镜像目录（参照模式11中 `.claude/agents/README.md` 需迁至 `.claude/README.md` 的案例），则需核对该规范是否适用于仓库根 README；如适用，需要调整文档的落盘位置/结构以符合校验规则。当前日志不足以区分"工具误报"与"真实路径规范要求"。

## 需要进一步确认的点
1. `eulerpublisher/update/container/app/update.py:273` 附近路径校验逻辑，以及 `eulerpublisher/update/container/app/format.py` 的路径解析（参考模式29 `format.py:101`），确认 `README.md` 为何被判为期望 `/README.md`。
2. 历史上是否存在同样只改根目录 `README.md` 的纯文档 PR 通过该门禁的记录；若无，则倾向确认为工具对 docs-only 变更的误报。
3. 该校验是否应跳过非镜像目录（如仓库根、`.claude/` 等）下的文档文件。

## 修复验证要求
不适用（本失败不涉及对第三方/上游源文件的正则 patch）。

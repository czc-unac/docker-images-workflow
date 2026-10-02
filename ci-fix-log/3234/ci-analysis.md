# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error
- 置信度: 高
- 知识库匹配: 模式11
- 新模式标题: (无)
- 新模式症状关键词: (无)

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
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范预检，路径校验逻辑）
- 失败原因: CI 的 appstore 发布规范预检将本 PR 的唯一变更文件 `README.md`（仓库根目录文档）作为待发布制品进行路径校验，判定其 `[Path Error]`，导致预检失败。

### 与 PR 变更的关联
- PR diff 仅修改仓库根目录 `README.md`（新增 12 行文档说明），无任何镜像目录/Dockerfile/meta 变更。
- 日志中 `Difference: ["README.md"]` 与 PR diff 完全一致，说明本次失败是由该 PR 的 README 改动**直接触发**（预检把该改动文件纳入发布路径规范检查）。
- 该 PR 为纯文档变更，不涉及镜像构建，因此失败点在于 CI 发布规范预检对“非镜像路径文件”的处理，而非代码/构建本身。

## 修复方向

### 方向 1（置信度: 高）
该 CI 失败为 appstore 发布规范预检对纯文档 PR 的路径校验触发：根目录 `README.md` 不属于任何镜像最小目录单元（`image-list.yml` 中登记的路径），因此被判定为路径错误。需确认预检是否应对“非镜像目录的文档类变更”予以豁免，或将该文档变更放置到符合预检期望的路径/位置。由于本 Agent 只做诊断，具体处置需结合 `update.py` 的路径校验规则确定。

### 方向 2（置信度: 中）
若预检规则确实要求所有变更文件都必须位于已登记的镜像目录内，则需要按规范调整该文档的提交位置（例如放入某个镜像目录下的文档路径），使其通过路径校验。

## 需要进一步确认的点
1. 查阅 `eulerpublisher/update/container/app/update.py` 第 273 行附近及“Path Error / expected path”校验函数的实现，确认其判定规则：是校验“变更文件必须属于 `image-list.yml` 登记的镜像目录”，还是存在其他路径前缀要求。
2. 确认该预检是否对纯文档/非镜像变更（如根目录 `README.md`）设有豁免白名单或跳过逻辑；若无，则本次失败属于预检规则与文档类 PR 的冲突。
3. 确认历史同类案例 `模式11` 中 `.claude/agents/README.md`、`.claude/README.md` 路径错误的处理结论，判断本次是否与之一致（README 文件因不在期望路径而被预检拒绝）。

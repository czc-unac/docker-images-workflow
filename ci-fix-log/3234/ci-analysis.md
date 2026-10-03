# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error
- 置信度: 中
- 知识库匹配: 模式11（YAML / 元数据文件错误 —— appstore 发布规范路径预检）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
2026-09-15 09:16:26,553-...update.py[line:356]-INFO: Difference: [
    "README.md"
]
2026-09-15 09:16:31,262-...update.py[line:273]-ERROR: There are some specification errors for releasing on appstore in this PR, please check as above.
+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范预检），触发项为仓库根目录 `README.md`
- 失败原因: 本 PR 仅修改仓库根目录 `README.md`，eulerpublisher 的 appstore 发布规范预检将变更文件 `README.md` 判为「路径不合规」（期望路径 `/README.md`），预检返回 FAILURE，整个流水线被标记为失败。

### 与 PR 变更的关联
- PR diff 仅对 `README.md` 增加 12 行（新增「提交 Issue 自动化生成镜像」说明），未新增/修改任何镜像目录（Dockerfile、meta.yml、image-info.yml）。
- CI 日志中 `Difference: ["README.md"]` 明确表明本次预检只看到 `README.md` 一个差异文件，失败完全由该文档改动触发。
- 也就是说，本次失败与镜像构建本身无关，是 appstore 发布规范预检对「根目录 README.md 变更」的路径校验不通过所致。该行为与知识库模式11 中的历史案例（PR #2512，`.claude/README.md` 路径不符合 appstore 规范预检）属同一类「README 路径预检」问题。

## 修复方向

### 方向 1（置信度: 中）
- 调整提交内容，避免让「仅文档类的顶层 README.md 变更」进入 appstore 发布规范预检的变更文件集合：
  - 方案 A：将本次新增的「自动生成新镜像」指南移动到仓库中预检允许的文档位置（例如独立文档目录，而非顶层 README.md），并相应更新引用；
  - 方案 B：若预检确实只接受镜像目录内的 `README.md`（如 `{场景}/{镜像}/{版本}/README.md`），则不要把该说明加到顶层 `README.md`，而是以不触发发布预检的形式（如单独的 docs 文件）承载。
- 依据：日志显示预检把 `README.md` 判定为 `[Path Error]`，说明顶层 README.md 不在该预检允许的 appstore 发布路径白名单内。

### 方向 2（置信度: 低）
- 若确认「修改顶层 README.md 不应触发 appstore 发布预检」，则这属于 CI 编排工具的误报，需要 eulerpublisher 侧的预检逻辑对非镜像类文件（纯文档）做过滤，而非在 PR 内改代码。此种情况下 Code Fixer 无需处理该失败，应联系 CI 运维调整工具。

## 需要进一步确认的点
- 日志未提供 eulerpublisher `update.py` 预检白名单/路径规则的完整代码，无法确认：
  1. 顶层 `README.md` 是否本就**不允许**出现在 appstore 发布预检的变更文件集合中（即是否应过滤纯文档文件）。
  2. 报错文案「The expected path should be /README.md」的具体判定逻辑（为何根目录的 `README.md` 会被判为路径错误）。
- 需确认 trigger 层为何对「仅文档变更」的 PR 仍执行 appstore 发布规范预检：是否是 trigger job 对 PR 类型判断有误，或对 `note` 触发的流水线一律执行预检。
- 需确认本仓库对顶层 `README.md` 更新的既有约定（历史上是否有同类文档 PR 通过 CI，或都需要特殊处理）。

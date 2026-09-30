# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（appstore 发布规范静态预检失败）
- 置信度: 中
- 知识库匹配: 新模式
- 新模式标题: 根目录文档路径校验
- 新模式症状关键词: Path Error, expected path should be /README.md, update.py, appstore specification, README.md

## 前置检查（日志与状态一致性）
`ci.logs` 末尾为 `Build step 'Execute shell' marked build as failure` 与 `Finished: FAILURE`，
**未**出现 `Finished: SUCCESS` / `Build successful`，因此不触发"证据不足"的前置终止条件，正常分析。

## 根因分析

### 直接错误
```text
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
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范校验）与 `:356`（变更差异统计）
- 失败原因: 本次 PR 的唯一变更文件为仓库根目录的 `README.md`，CI 的 appstore 发布规范预检对该文件报 `[Path Error] The expected path should be /README.md`，判定规范错误并以 FAILURE 结束。

### 与 PR 变更的关联
- PR diff 显示唯一改动为 `README.md`（新增"自动化新增镜像 Issue 指南"段落），`update.py` 统计到的差异集合恰为 `["README.md"]`，与报错项完全对应。即 CI 失败确实由该 PR 触发。
- 但报错语义存在矛盾：变更文件本身就在仓库根目录，校验期望路径也是 `/README.md`，却被判为路径错误。这说明该校验对"根目录文档类文件"的路径规范化/归属判定可能存在误判，或该校验只会放行属于具体镜像目录的文件（非镜像目录的根级文档被当作 spec 文件校验）。

## 修复方向

### 方向 1（置信度: 中）
该失败源于 appstore 发布规范预检把根目录 `README.md` 纳入镜像发布规范校验范围并报路径错误，而非 Dockerfile / 构建逻辑问题。docs-only 的本 PR 无需修改镜像构建内容；如需通过门禁，应将纯文档说明移出会触发该校验的位置，或由仓库维护方确认根目录 README 变更是否应豁免该校验。（不在此处给出具体实现。）

### 方向 2（置信度: 低）
若项目规范确实要求文档类文件放在某个被校验脚本认可的路径，则应把新增指南内容放到校验脚本期望的 README 位置/目录，而不是根 README。

## 需要进一步确认的点
1. `eulerpublisher/update/container/app/update.py:273` 附近 appstore 路径校验的判定逻辑——特别是 `[Path Error]` 的期望路径是如何计算/规范化得到的。
2. 该校验是否对所有根目录级、非镜像目录的文件都报同类路径错误（即是否为 docs-only PR 的通用限制）。
3. 对照历史模式 11 中 `.claude/README.md` 路径校验失败案例，确认本场景属于同一类"文档路径校验"，还是根目录 README 的独立变体。
4. PR 目标分支/仓库中根 README 是否本就存在且格式被接受；若本 PR 之前根 README 从未被该校验扫描过，则需确认触发条件。

## 修复验证要求
不适用（本失败不涉及对第三方/上游源文件使用正则 patch 的场景）。

# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（CI appstore 发布规范 / 路径校验不通过）
- 置信度: 中
- 知识库匹配: 模式11（YAML / 元数据文件错误 — appstore 路径校验类）
- 新模式标题: （不适用）
- 新模式症状关键词: （不适用）

## 根因分析

### 直接错误
```
2026-09-15 09:16:26,553-.../update/container/app/update.py[line:356]-INFO: Difference: [
    "README.md"
]
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
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范校验逻辑），触发点为 `:356` 输出的 PR 差异清单 `["README.md"]`
- 失败原因: appstore 发布前的规范校验把本次 PR 的唯一变更文件 `README.md` 判定为“路径不符合预期”，要求其位于 `/README.md`，从而抛出 `specification errors for releasing on appstore`。

### 与 PR 变更的关联
本次 PR 为纯文档改动，仅在仓库根 `README.md` 的“镜像发布指南”与“关于门禁检查的说明”之间新增了一段“通过 Issue 申请新增镜像”的说明（新增 12 行、删除 0 行），未新增/修改任何镜像目录、Dockerfile 或元数据文件。

值得注意的是：diff 中该文件的路由为 `--- a/README.md` / `+++ b/README.md` / `new_path: README.md`，本身已经位于仓库根目录，与校验要求的 `/README.md` 在语义上一致。但 CI 仍报 `[Path Error]`。因此该失败有两种可能：
1. appstore 校验工具在“PR 仅包含文档变更、没有镜像目录”的场景下错误地把 `README.md` 归入镜像路径校验，属于工具侧对纯文档 PR 的误判（与代码改动无因果关系）；
2. 校验工具确实要求特定形式的路径（如 `/README.md` 绝对形式）而当前 diff 表示形式触发了不一致，但这需要查看 `update.py` 的实现才能确认。

无论哪种情况，**日志只能证明失败发生在 appstore 规范校验阶段，无法从现有日志完整还原工具判定路径的具体依据**，故置信度定为“中”。

## 修复方向

### 方向 1（置信度: 中）
确认 appstore 规范校验对“仅文档变更的 PR”是否应触发。如果工具对不含镜像目录的纯文档 PR 存在误判，则属于 CI/工具侧问题（接近 infra-error），需要调整校验逻辑或触发条件，而非修改文档内容。

### 方向 2（置信度: 中）
若校验确实要求文档说明必须放在某个特定位置/文件（例如申请镜像的指南应写入贡献说明或指定路径的 README），则应将新增说明移动到校验所期望的位置，避免改动根 `README.md` 触发路径检查。

## 需要进一步确认的点
- 查看 `eulerpublisher/update/container/app/update.py:273` 与 `:356` 的具体实现，确认：
  - 校验工具是如何从 `Difference` 列表推导“期望路径”的（为何期望 `/README.md` 而实际判定失败）；
  - 该 appstore 规范校验是否对“仅修改 README.md、无镜像目录”的 PR 做了特殊处理。
- 确认 CI 期望路径 `/README.md` 中的前导 `/` 的语义（是相对镜像根目录，还是相对仓库根目录），以及 diff 中 `README.md` 与之的对应关系。
- 确认同类纯文档 PR（历史案例 PR #2308、#2489、#2516、#2894）是否通过该 appstore 校验，以判断这是工具误判还是本 PR 特有问题。
- 确认本次 PR 是否应只改文档；若镜像申请指南有官方指定的落盘位置，需按规范放置。

## 修复验证要求
本失败位于 CI 工具（eulerpublisher）的 appstore 规范校验阶段，现有日志无法完全证明是文档内容问题还是工具误判。在按“方向 2”调整文档位置前，code-fixer 必须先确认校验工具对 `README.md` 路径的期望语义（见“需要进一步确认的点”），不得直接假定移动/删改文档即可修复。若确认属于工具对纯文档 PR 的误判，则不应对本 PR 的文档内容做无依据改动。

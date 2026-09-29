# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（appstore 发布规范/路径预检失败）
- 置信度: 中
- 知识库匹配: 模式11（YAML / 元数据文件错误 / appstore 路径校验失败）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
2026-09-15 09:16:26,553-...update.py[line:356]-INFO: Difference: [
    "README.md"
]
2026-09-15 09:16:31,258-...update.py[line:222]-INFO: Clone https://gitcode.com/qq_42020325/****-docker-images.git successfully.
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
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范预检报错），被检文件为仓库根目录 `README.md`
- 失败原因: eulerpublisher 的 appstore 发布规范预检判定本次 PR 中 `README.md` 的路径不符合规范，期望路径为 `/README.md`，因此预检表项置为 `FAILURE`，构建脚本据此将构建标记为失败。

### 与 PR 变更的关联
高度相关。日志中 `Difference: ["README.md"]` 表明本次 PR 的**唯一变更文件就是根目录 `README.md`**（与 `pr.diff` 一致：仅在 README.md 第 200 行附近新增了 12 行「自动化新增镜像 Issue 流程」文档）。该变更正是被 appstore 预检直接判定为路径错误的对象，即失败由本 PR 的文件变更触发，而非构建/编译/测试阶段的代码缺陷。

### 影响范围评估
局部问题。变更仅涉及根目录文档文件，未触及任何镜像目录、Dockerfile、`meta.yml` 或 `image-list.yml`，失败被局限在发布规范预检环节。

## 修复方向

### 方向 1（置信度: 中）
appstore 预检将根目录 `README.md` 视为发布规范路径违规。需先确认该预检对根目录 README 的路径规则：
- 若文档类变更本不应进入「appstore 发布检查」流程，则应从流程/触发侧排除对根目录文档的检查（如按 note 触发的发布会话不应针对纯文档 PR 执行该规范检查）；
- 若规范要求 README 必须位于镜像最小目录单元之下，则需将新增说明迁移到符合规范的路径，或拆分为独立文档文件后提交。

具体以 `update.py` 中生成该检查表的逻辑为准，不预设结论。

### 方向 2（可选，置信度: 低）
存在 eulerpublisher 路径归一化问题的可能：diff 侧路径 `README.md`（无前导斜杠）与期望值 `/README.md`（带前导斜杠）比较不一致，导致误报。若确认如此，则属 CI 工具侧缺陷（infra），Code Fixer 无需修改代码，应反馈给流水线维护方。

## 需要进一步确认的点
1. `eulerpublisher/update/container/app/update.py:273` 附近逻辑及其生成「Check Items / Description / Check Result」表的函数，确认 `README.md` 路径校验的实际规则与「/README.md」这一期望值的计算方式。
2. 该预检是否维护「允许变更文件」白名单，仓库根 `README.md` 是否在其中；以及该流水线是否只应针对镜像发布类 PR 运行。
3. 本次构建的触发方式（日志为 `trigger by note`，由上游 `multiarch/openeuler/trigger/openeuler-docker-images` 触发）是否会对纯文档 PR 错误地执行 appstore 发布规范检查。

## 修复验证要求
本次失败不涉及对第三方/上游源文件的正则 patch，无需从上游仓库拉取验证。若修复方向选择「调整预检规则」，应在提交前用本 PR 相同的纯文档变更复现该预检流程，确认 `README.md` 不再被标记为 `FAILURE`。

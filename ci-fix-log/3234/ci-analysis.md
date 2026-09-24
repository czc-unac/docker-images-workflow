# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（CI 规范/预检校验失败）
- 置信度: 中
- 知识库匹配: 模式11
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
2026-09-15 09:16:26,553 - .../eulerpublisher/update/container/app/update.py[line:356] - INFO: Difference: [
    "README.md"
]
2026-09-15 09:16:31,258 - .../eulerpublisher/update/container/app/update.py[line:222] - INFO: Clone https://gitcode.com/qq_42020325/****-docker-images.git successfully.
2026-09-15 09:16:31,262 - .../eulerpublisher/update/container/app/update.py[line:273] - ERROR: There are some specification errors for releasing on appstore in this PR, please check as above.
+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范预检逻辑）
- 失败原因: 本 PR 唯一改动文件为仓库根目录 `README.md`，CI 差异检测得到 `Difference: ["README.md"]`，随后 appstore 发布规范预检对该文件执行路径校验，判定 `[Path Error] The expected path should be /README.md`，触发 `update.py:273` 的 specification error 并使构建失败。

### 与 PR 变更的关联
- PR diff 仅新增 12 行文档内容，文件为 `README.md`（`a/README.md` → `b/README.md`，非新增、非重命名）。
- CI 日志中的差异列表恰为 `["README.md"]`，与本次改动完全一致，说明失败由本 PR 的文档改动直接触发（并非无关的基础设施抖动）。
- 该 job 为 x86-64 架构构建链上的初始化/预检阶段（`ecs-build-docker-x86-hk`），在执行 Docker 构建前即因规范校验退出。日志末尾为 `Finished: FAILURE`（非 SUCCESS），因此不属于“成功日志 + ci_failed 标签”的证据不足场景，校验失败可确认。

## 修复方向

### 方向 1（置信度: 中）
将本次新增的“自动化新增镜像请求指南”文档内容从仓库根 `README.md` 中移出，放到 CI appstore 规范预检允许/可识别的路径或既有文档文件内，使变更文件不再以根 `README.md` 身份进入镜像路径校验（参考知识库中 `.claude/*/README.md` 路径校验失败的同类处置）。

### 方向 2（置信度: 中）
若确认根 `README.md` 属于合法文档载体（预期路径即 `/README.md`），则此失败更可能是 `update.py` 预检逻辑对“纯文档 PR / 根级 README.md”处理不当（路径归一化或豁免规则缺失），应作为 CI 工具侧问题处理，而非修改 PR 文档内容。此方向需先完成下方“需要进一步确认的点”。

## 需要进一步确认的点
- 需查阅 `eulerpublisher/update/container/app/update.py:273` 及其调用的路径校验函数，确认 `[Path Error] The expected path should be /README.md` 的判定规则：为何根目录 `README.md` 会被判定为路径错误（是要求镜像文件必须位于 `{image}/{version}/{os}/` 结构下，还是路径归一化 bug）。
- 确认该 appstore 预检是否对“仅修改仓库根说明文档（README.md）”的 PR 有豁免机制；若无，纯文档 PR 是否都无法通过该门禁。
- 确认 `update.py:356` 打印的 `Difference` 是否仅包含本次 PR 改动文件，以排除其他上游/基线差异干扰。

## 修复验证要求
本次修复方向不涉及“修改正则匹配外部/上游源文件”，无需按该场景提供上游拉取验证。但若采用方向 1 调整文档位置，code-fixer 需在执行前确认 CI appstore 预检实际接受的路径集合；若采用方向 2，需先获取 `update.py` 路径校验逻辑并复现判定，不能直接假设为工具缺陷。

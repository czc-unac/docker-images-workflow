# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error
- 置信度: 中
- 知识库匹配: 模式11
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
2026-09-15 09:16:26,553 .../update/container/app/update.py[line:356]-INFO: Difference: [
    "README.md"
]
2026-09-15 09:16:31,258 .../update/container/app/update.py[line:222]-INFO: Clone https://gitcode.com/qq_42020325/****-docker-images.git successfully.
2026-09-15 09:16:31,262 .../update/container/app/update.py[line:273]-ERROR: There are some specification errors for releasing on appstore in this PR, please check as above.
+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: eulerpublisher `update/container/app/update.py:273`（appstore 发布规范预检），触发点为本次 PR 唯一改动文件 `README.md`
- 失败原因: 本次 diff 仅修改根目录 `README.md`（新增"新增应用镜像可通过 Issue 由流水线自动生成"的文档说明）。CI 在 appstore 发布规范预检阶段计算变更差异，得到 `Difference: ["README.md"]`，随后对该文件做发布路径校验并判定 `[Path Error] The expected path should be /README.md`，预检失败导致 job 失败。

### 与 PR 变更的关联
- 直接关联：`ci.logs` 中 `Difference: ["README.md"]` 与 `pr.diff` 中唯一改动文件 `a/README.md / b/README.md` 完全一致，可确认失败由本次对根目录 README.md 的修改触发。
- 注意：日志末尾为 `Finished: FAILURE`（非 SUCCESS），因此不属于"日志成功但状态失败"的证据不足场景，可继续分析。
- 语义疑点：校验信息 "The expected path should be /README.md" 与被检文件路径（本身就是根目录 `README.md`）表面矛盾，说明该预检对"非镜像类文件（文档）混入发布变更集"的处理逻辑不清晰，无法仅凭日志 100% 断定是代码问题还是 CI 预检误判。

## 修复方向

### 方向 1（置信度: 中）
若 CI 预期根目录 README.md 的变更不应进入 appstore 发布规范校验，则这是预检工具（eulerpublisher `update.py`）对文档类变更的误判。修复思路：调整/拆分变更，使文档改动不落入该发布校验的差异集（例如将文档并入被 CI 识别为文档/忽略的路径，或在 CI 侧为根目录 README.md 增加豁免），不修改任何镜像构建文件。

### 方向 2（置信度: 中）
若该预检要求被变更文件必须落在某个合法镜像目录结构内（两级 `{image-version}/{os-version}/`），则根目录 README.md 天然不满足。修复思路：将本次新增文档内容放置到 CI 校验认可的文档路径下，或在提 PR 时仅提交被发布框架允许的文件类型。

## 需要进一步确认的点
- 需确认 `eulerpublisher update.py` 的 appstore 发布规范校验（约 line 273 前后）对 `README.md` 的判定规则：为何对路径已是根目录的 `/README.md` 仍报 `[Path Error] The expected path should be /README.md`。
- 需确认本次失败是否为"纯文档 PR 误触发发布预检"的既有 CI 行为（可参考模式11历史案例 PR #2512 中 `.claude/README.md` 等路径预检失败），还是本次 README.md 新增内容本身包含被误识别为镜像条目的文本。
- 需确认历史同类纯文档 PR（如模式19案例 #2308 `AI/diskann/README.md`）当时是否也触发该预检、最终如何处理。
- 若确认为 CI 预检工具缺陷或豁免规则缺失，应标记为 infra-error 并由 CI 维护方处理，而非修改本 PR 的文档内容。

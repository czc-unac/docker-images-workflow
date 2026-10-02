# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（appstore 发布规范/路径静态校验失败）
- 置信度: 低
- 知识库匹配: 模式11（近似，但根因不同）
- 新模式标题: （见下方说明；若判定为独立新模式，标题：文档PR路径误判）
- 新模式症状关键词: appstore, Path Error, README.md, update.py, specification errors

> 前置检查：`ci.logs` 末尾为 `Finished: FAILURE`（“Build step 'Execute shell' marked build as failure … Finished: FAILURE”），
> 并非 `Finished: SUCCESS`，因此**不属于**“日志成功但 PR 失败”的 infra-error 场景，继续分析。

## 根因分析

### 直接错误
```
### Error lines (newest first)
2026-09-15 09:16:31,262-.../eulerpublisher/update/container/app/update.py[line:273]-ERROR: There are some specification errors for releasing on appstore in this PR, please check as above.
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |

### Build tail（节选）
2026-09-15 09:16:26,553-.../update.py[line:356]-INFO: Difference: [
    "README.md"
]
2026-09-15 09:16:31,258-.../update.py[line:222]-INFO: Clone https://gitcode.com/qq_42020325/****-docker-images.git successfully.
2026-09-15 09:16:31,262-.../update.py[line:273]-ERROR: There are some specification errors for releasing on appstore in this PR, please check as above.
+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范校验），触发文件为仓库根目录 `README.md`
- 失败原因: CI 在镜像发布规范预检阶段，把本次 PR 变更的 `README.md` 判定为“发布制品”，并报 `[Path Error] The expected path should be /README.md`，导致规范校验整体 FAILURE。

### 与 PR 变更的关联
- 强关联：日志 `update.py[line:356]-INFO: Difference: ["README.md"]` 表明本次 PR 的变更集**仅包含 `README.md`**，而校验失败项恰好就是 `README.md`，因此失败由本次 PR 的文档改动直接触发。
- 但报告内容存在自相矛盾：`README.md` 本身即位于仓库根目录，按报错文案“expected path should be /README.md”本应通过。这说明该失败**很可能不是文档内容本身的问题**，而是预检工具/流水线对“纯文档 PR”的误判或路径归一化问题。

## 失败分类判断说明
- 主判为 `lint-error`（appstore 发布规范/路径静态校验）。
- 若确认该报错为流水线对纯文档改动的误触发（即校验逻辑不应作用于根 `README.md`），则应归为 `infra-error`，Code Fixer 无需改动文档。两种判断均缺乏直接证据，故置信度为低。

## 修复方向

### 方向 1（置信度: 低）
先确认 appstore 发布规范校验的触发条件：若该校验仅应在涉及镜像文件（`Dockerfile` / `meta.yml` / `image-info.yml` / `image-list.yml`）变更时执行，则纯文档 PR（仅改根 `README.md`）不应进入该检查。若属此情况，本失败为流水线误判，文档无需修改。

### 方向 2（置信度: 低）
若校验逻辑确实要求根 `README.md` 参与，则需核对 `update.py` 中路径归一化/比对逻辑（为何文件已在 `/README.md` 仍报 `[Path Error]`），确认为工具缺陷后按 CI 侧问题处理，而非修改 PR 文档。

## 需要进一步确认的点
1. `eulerpublisher/update/container/app/update.py` 中 line 273 附近及路径校验函数（`_parse_*` / path check）的实际判定逻辑，明确 `[Path Error] The expected path should be /README.md` 的触发条件。
2. 传入校验的“实际路径”归一化结果，确认根目录 `README.md` 为何被判定不符合 `/README.md`。
3. appstore 发布规范校验是否对“仅文档变更”的 PR 有豁免/排除规则。
4. 该 job 为 x86-64 架构 job（`Building remotely on ecs-build-docker-x86-hk`），但失败发生在入库预检而非编译；确认是否还有其他架构 job 的日志可佐证同一错误。

## 修复验证要求
- 本报告置信度为“低”，且失败信息自相矛盾。Code Fixer **不得**据此直接修改 `README.md` 内容。
- 必须先完成上述确认点 1–3，确认校验逻辑究竟期望何种路径，再决定是否需要改动。
- 本失败不涉及对第三方/上游源文件的正则 patch，无需执行外部仓库正则匹配验证。

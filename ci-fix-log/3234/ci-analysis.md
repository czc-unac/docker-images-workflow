# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（CI 应用镜像发布规范/路径预检失败）
- 置信度: 中
- 知识库匹配: 模式11（YAML / 元数据 / 路径校验失败）
- 新模式标题: （不适用，匹配既有模式）
- 新模式症状关键词: （不适用）

## 根因分析

### 直接错误
```text
2026-09-15 09:16:31,262-.../eulerpublisher/update/container/app/update.py[line:273]-ERROR: There are some specification errors for releasing on appstore in this PR, please check as above.
+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

前置信息：
```text
.../update.py[line:356]-INFO: Difference: ["README.md"]
.../update.py[line:222]-INFO: Clone https://gitcode.com/qq_42020325/****-docker-images.git successfully.
```

### 根因定位
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（应用镜像发布规范检查），被校验对象为本 PR 变更文件 `README.md`
- 失败原因: CI 的应用镜像（appstore）发布规范预检在比对变更文件时，将 `README.md` 判定为 `[Path Error]`，要求其路径为 `/README.md`，规范校验不通过，导致 job 失败（`Finished: FAILURE`）。

### 与 PR 变更的关联
本 PR 为纯文档改动，仅向仓库根 `README.md` 新增 12 行“通过 Issue 自动新增应用镜像”的说明（`pr.diff` 中 `old_path/new_path` 均为 `README.md`）。CI 通过 `Difference: ["README.md"]` 明确识别到该变更文件，随后对该文件执行发布规范路径校验并失败。因此，**失败由本 PR 修改根 `README.md` 这一动作直接触发**，而非下游架构构建错误。

## 修复方向

### 方向 1（置信度: 中）
调整文档放置方式，使其不落入应用镜像发布规范预检的路径校验范围。根 `README.md` 被该检查纳入“应用镜像发布”流程，而本次改动与任何镜像目录无关；可考虑将新增内容放入 CI 未纳入镜像路径校验的文档位置，或确认是否应修改根 `README.md`。（具体落点需以 CI 校验规则为准，本报告不提供代码。）

### 方向 2（置信度: 低）
若确认根 `README.md` 的预期路径确为 `/README.md`（即当前文件位置本就正确），则该 `[Path Error]` 可能来自校验器对根级文档的路径匹配缺陷（如相对路径与绝对路径字符串不一致导致的误报）。此时失败属于 CI 预检工具的缺陷，需由 CI/工具侧修复，PR 侧无代码问题。

## 需要进一步确认的点
1. CI 规范检查（`eulerpublisher/update/container/app/update.py`）对根目录 `README.md` 的路径校验规则：为何 `README.md` 被要求为 `/README.md`，以及该规则是否允许在应用镜像发布流程中修改仓库根 README。
2. 该检查是否对所有纯文档 PR 都会执行，还是仅当变更文件命中某类路径（如 `README.md`）时才触发；需确认是否存在“文档改动被误纳入 appstore 发布检查”的既有行为。
3. 历史相近案例（模式11，PR #2512 的 `.claude/agents/README.md` 路径校验失败）中，README 被要求上移一级；需确认本 PR 是否存在类似的“期望路径位移”语义，而非单纯的相对/绝对路径写法差异。
4. 由于日志中未展示 `update.py` 路径校验函数的完整输入（仅给出结论行与 `Difference: ["README.md"]`），无法确认校验时使用的路径基准目录（仓库根 vs 镜像根），需结合该工具源码或同仓库其他 README 案例进一步确认。

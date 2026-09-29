# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（CI 发布规范 / 路径校验预检失败，非编译、非测试失败）
- 置信度: 中
- 知识库匹配: 模式11（元数据 / 路径一致性校验失败）
- 新模式标题: （不适用，已匹配现有模式）
- 新模式症状关键词: （不适用）

## 根因分析

### 直接错误
```text
2026-09-15 09:16:26,553 update.py[line:356]-INFO: Difference: [
    "README.md"
]
2026-09-15 09:16:31,258 update.py[line:222]-INFO: Clone https://gitcode.com/qq_42020325/****-docker-images.git successfully.
2026-09-15 09:16:31,262 update.py[line:273]-ERROR: There are some specification errors for releasing on appstore in this PR, please check as above.
+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

说明：日志末尾为 `Finished: FAILURE`（不是 `Finished: SUCCESS`），失败状态与日志一致，前置一致性检查通过，可继续分析。

### 根因定位
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范校验报错），
  差异文件来源见 `update.py:356`（`Difference` 计算）。日志中实际被校验的文件为 `README.md`。
- 失败原因: CI 在执行 appstore 发布规范预检时，将本次 PR 的唯一变更文件 `README.md` 判为路径不合规，
  提示期望路径为 `/README.md`，规范校验表返回 `FAILURE`，整个 job 被标记为失败。
- 失败发生在构建/测试之前的规范预检阶段，未进入任何镜像的 Docker build 或功能测试。

### 与 PR 变更的关联
该 PR 为纯文档变更，diff 仅在根目录 `README.md` 新增 12 行"自动化新增应用镜像申请指南"，删除 0 行。
CI 的 `Difference` 输出恰为 `README.md`，即被规范校验拦截的对象正是本次 PR 修改的这个文件。
因此，失败由本 PR 的文档改动直接触发（变更文件路径未通过 appstore 规范校验）。

## 影响范围评估
局部问题，位于 CI 发布规范预检层，与镜像 Dockerfile 构建、功能测试无关；不涉及多架构构建失败。

## 修复方向

### 方向 1（置信度: 中）
该预检针对"发布到 appstore"的镜像相关文件做路径规范校验。本 PR 仅改动根目录 `README.md`，
需确认根目录 README 是否属于该校验允许的路径范畴；若不允许，应将新增文档放置到校验规则接受的目录/位置，
或将文档内容并入已有的规范文档中，避免单独修改触发路径校验的 `README.md`。

### 方向 2（置信度: 中/低）
校验提示"期望路径为 `/README.md`"而实际差异路径为 `README.md`（缺少前导 `/`），
可能是 `update.py` 路径规范化逻辑与 diff 输出格式不一致导致的误报（CI 工具层问题）。
若确认根目录 README 本应被允许，则该失败属 `infra-error`，Code Fixer 无需修改仓库代码，
需由 CI / eulerpublisher 维护方修复路径校验逻辑。

## 需要进一步确认的点
1. `eulerpublisher/update/container/app/update.py:273` 及其调用的路径校验函数，对 `README.md`
   的"期望路径"判定规则（为何期望 `/README.md`，实际输入为 `README.md`）。
2. appstore 规范校验是否要求文档必须位于各镜像最小目录单元内（如 `{image}/README.md`），
   根目录 `README.md` 是否在白名单内。
3. `update.py:356` 计算 `Difference` 的文件路径格式（是否带前导 `/`），以判断是否为路径归一化缺陷。
4. 历史上根目录 `README.md` 的文档 PR 是否同样触发该预检失败，用于区分"规范要求"与"CI 工具误报"。

## 修复验证要求
本修复方向不涉及对第三方/上游源文件使用正则 patch，无需执行上游文件拉取验证。
若最终判定为方向 1（需调整文档位置），Code Fixer 应先确认规范校验所接受的路径，再移动/合并文档；
若判定为方向 2（工具层误报），应标记为 `infra-error`，不在本仓库内做代码修改。

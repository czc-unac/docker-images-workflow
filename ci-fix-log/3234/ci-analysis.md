# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: `lint-error`（CI appstore 发布规范 / 路径静态预检失败）
- 置信度: 中
- 知识库匹配: 模式11（YAML / 元数据文件错误——含 README 路径校验失败场景）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

> 前置检查：`ci.logs` 的 Build tail 末尾为 `Finished: FAILURE`，未出现 `Finished: SUCCESS` / `Build successful`，因此不属于"成功日志 + 失败状态"的证据不足场景，继续进行根因分析。

## 根因分析

### 直接错误
```text
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
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范预检；变更集来自同文件 line:356 打印的 `Difference: ["README.md"]`）
- 失败原因: 本次 PR 的变更文件只有 `README.md`，CI 的 appstore 发布规范预检对 `README.md` 做路径校验时判定 `[Path Error] The expected path should be /README.md`，预检不通过（该 job 在 `multiarch/openeuler/x86-64/openeuler-docker-images` 工作空间执行）。

### 与 PR 变更的关联
直接相关。`pr.diff` 显示该 PR 唯一改动是在仓库根目录 `README.md` 中新增"新增应用镜像可通过 Issue 自动化流水线完成"的文档段落（+12 行）。日志中 `update.py` 的 `Difference` 列表恰好只有 `"README.md"`，随后即报出针对 `README.md` 的 path error，说明失败由本次对根目录 `README.md` 的修改触发。这是一个纯文档改动，不涉及任何镜像 Dockerfile 或构建逻辑，但被 appstore 发布规范校验流程纳入了路径检查范围。

## 修复方向

### 方向 1（置信度: 中）
按 appstore 预检对 README 的路径要求调整文档落点：确认 CI 期望的 README 目标路径（日志提示 `/README.md`），使本次文档改动位于被校验工具接受的路径/文件中；若该预检不接受对仓库根级 `README.md` 的直接修改，则应将新增的"new-image 指南"放到校验允许的位置（例如对应镜像目录下的文档或仓库约定的 README 位置）。不提供具体代码。

### 方向 2（置信度: 低）
若确认仓库根目录 `README.md` 本就是合法路径、且实际文件位置确为 `/README.md`，则该 `[Path Error]` 可能是校验脚本对路径前导 `/` 归一化处理的缺陷（工具侧问题）。此方向需要在代码库/校验脚本中核对 `update.py` 的路径比对逻辑后才能定论，Code Fixer 不应在未核实前直接改动。

## 需要进一步确认的点
1. `eulerpublisher/update/container/app/update.py` 第 356 行与第 273 行附近：`Difference` 文件列表如何被映射为"期望路径"，以及 `README.md` 被判为 `[Path Error] expected path /README.md` 的确切比对规则（是否存在前导 `/` 归一化问题）。
2. 该仓库 appstore 发布规范校验的允许路径白名单：根目录 `README.md` 是否属于合法变更路径；若合法，为何本次仍报 path error。
3. 校验触发条件：是否为"变更集中不含任何镜像目录文件"时对残留文件（如根 README）统一报错。
4. 参照知识库模式11中 PR #2512 的 README 路径校验案例，确认本仓库对应的期望 README 路径约定。

## 修复验证要求
本失败方向不涉及"修改正则去匹配第三方/上游源文件"，因此不强制上游取源验证。但鉴于置信度为"中"：
- Code Fixer 在提交前必须先在仓库中核对上述 `update.py` 的路径校验逻辑与允许路径白名单，确认根目录 `README.md` 的合法目标路径，禁止在未核实的情况下臆断路径。
- 若最终确认根 `README.md` 修改本身合法、失败源于校验脚本路径比对缺陷，则应判为工具侧问题并转 infra 处理，不应改动文档内容。

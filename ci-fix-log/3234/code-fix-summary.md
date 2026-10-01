# 修复摘要

## 修复的问题
无需代码修改：PR 仅改动仓库根目录的 `README.md`，被 EulerPublisher 的 appstore 发布规范预检当作镜像文档处理并误报路径错误，属于 CI 校验工具对 docs-only 根目录 README 变更的误报（false positive），无法通过修改 `README.md` 内容消除。

## 修改的文件
（无）

## 修复逻辑
1. **复现并确认根因（外部校验工具缺陷）**：已从上游镜像仓库 `openeuler-mirror/eulerpublisher@master` 拉取 CI 实际使用的 `update/container/app/format.py` 与 `update/container/app/update.py`，并在本地复现了校验逻辑：
   - `check_report()` 遍历改动文件时，按文件 basename 判定类型：`README.md` → `file_type = "README"`，命中 `DOC_FILES_PATH_FORMAT`（键包含 `README`、`meta`、`doc`、`image-info` 等），因此**不会跳过**根目录 README。
   - `parse_image_prefix("README.md")` 因路径只有一段（`len(contents) == 1`）返回 `("", "")`，`prefix` 为空串。
   - `_check_all_file_paths()` 用 `DOC_FILES_PATH_FORMAT["README"] = "{0}/README.md"` 拼接，`"" + "/README.md"` 得到绝对路径 `/README.md`，而该路径在 CI 工作目录的临时克隆目录下不存在，于是报 `[Path Error] The expected path should be /README.md`。
   - 本地模拟验证：`README.md` → 期望路径 `/README.md`（不存在）；`README.en.md` 同样被识别为类型 `README` 并期望 `/README.md`（同样失败）。
2. **为何不改 README.md**：该校验完全基于**文件路径**（`os.path.exists(correct_path)`），不读取文件内容。根目录任何 `README.*`（含 `README.md`、`README.en.md`）都会落入同一分支、期望一个带前导 `/` 的绝对路径。因此对 `README.md` 做任何内容层面的最小改动（增/删/改文档）都无法改变 `change_file` 的路径，也就无法让该检查通过。
3. **与允许修改范围的冲突**：本次仅允许修改原始 PR 涉及的 `README.md`，且禁止新增/删除文件、禁止改 CI 工具。将 README 改名/删除或改动外部 `format.py` 均超出授权范围且会破坏仓库约定，不属于最小化修复。
4. **结论**：分析报告“修复方向 1（置信度 中）”成立——该失败为 CI appstore 发布规范预检对“根目录 README 文档变更”的判定缺陷，与镜像发布无关。正确处理是判定为 CI 误报、不修改仓库文件，由维护者/工具侧修复（例如让 `format.py` 在 `prefix` 为空时跳过根目录文件，或用 `os.path.join` 替代字符串拼接以避免产生 `/README.md`）。故本次不做代码修改。

## 潜在风险
无。未改动任何仓库文件，不会引入构建/发布/文档回归。需注意：在 CI 工具侧修复前，任何只改动仓库根目录 `README.md`/`README.en.md` 的纯文档 PR 都会重复触发同一误报。
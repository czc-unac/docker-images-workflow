# 修复摘要

## 修复的问题
本 PR 仅修改仓库根目录 `README.md`（新增"通过 Issue 自动新增应用镜像"指南），CI 的 appstore 发布规范预检对其报 `[Path Error] The expected path should be /README.md`。经定位，这是 CI 工具 `eulerpublisher` 的路径校验缺陷（infra-error），并非 PR 内容问题，代码侧无需修改。

## 修改的文件
- 无。未对 `README.md`（或任何其他文件）做修改。

## 修复逻辑
分析报告存在两种方向（infra 误报 vs. 调整文档落点）。为确定根因，已从上游 `openeuler-mirror/eulerpublisher`（master 分支）拉取并审阅了实际执行该检查的源文件：

- `update/container/app/update.py:273` 的 `check_code()` 调用 `format.check_report(self.change_files)`
- `update/container/app/format.py` 中的 `check_report()` / `_check_all_file_paths()` / `parse_image_prefix()`

根因链路（可复现）：
1. `change_files = ["README.md"]`。
2. `check_report()` 取 `file_type = "README.md".split(".")[0] == "README"`，该键存在于 `DOC_FILES_PATH_FORMAT`（`"README": "{0}/README.md"`），因此被纳入校验。
3. `parse_image_prefix("README.md")` 中 `file.split("/")` 长度为 1，函数直接 `return "", ""`，得到空前缀 `prefix = ""`。
4. `_check_all_file_paths()` 以该空前缀格式化，得到 `correct_path = "/README.md"`（注意是文件系统绝对根路径，而非仓库根）。
5. `os.path.exists("/README.md")` 恒为 False，于是返回 `[Path Error] The expected path should be /README.md`，`fail_count` 增加，`check_code()` 返回 1，job 失败。

本地按上述逻辑仿真验证结果一致：`prefix='' correct_path='/README.md' exists=False`，而仓库根 `README.md` 实际存在。即：**任何修改仓库根目录 `README.md` 的 PR 都会触发该误报**，与本次新增的 12 行文档内容无关。

由于该检查只依据"变更文件名"判定，且 `README.md` 的内容无法改变其判定路径，因此不存在对 `README.md` 的最小化修改能使校验通过；根因位于 `eulerpublisher` 侧，源码不在本仓库，超出允许修改的文件范围（`pr.changed_files = ["README.md"]`）。按流程约定，此属 infra-error，不强行修改代码。

建议的 CI 工具修复（供工具维护方参考，不在本仓库实施）：在 `format.check_report()` 的 Step 1 中，当 `parse_image_prefix()` 返回空前缀（即文件位于仓库根目录）时跳过校验，或在 `_check_all_file_paths()` 前对 `prefix == ""` 做过滤；可避免所有根目录文档（如 `README.md`）被误判。

## 潜在风险
无（未改动任何源码/文档）。需注意：在 CI 工具修复前，任何仅修改仓库根目录 `README.md` 的 PR 都会继续出现该 `[Path Error]` 误报，需要重跑/豁免该检查或等待工具侧修复。
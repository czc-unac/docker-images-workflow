# 修复摘要

## 修复的问题
无需修改代码：CI 失败是 EulerPublisher 的 appstore 发布规范校验工具将仓库根目录 `README.md` 误判为镜像级 `README` 文档导致的工具缺陷（infra-error），仅修改 `README.md` 的内容无法消除该失败。

## 修改的文件
- 无（未对源码做任何改动）

## 修复逻辑
根因已通过拉取上游源文件核实，确认与本次 PR 的文档内容无关：

1. CI 在 `eulerpublisher/update/container/app/format.py` 的 `check_report()` 中逐项遍历变更文件，用 `file_type = change_file.split("/")[-1].split(".")[0]` 得到类型。根目录 `README.md` 得到的 `file_type` 为 `README`。
2. `DOC_FILES_PATH_FORMAT` 中包含键 `"README": "{0}/README.md"`，因此根目录 `README.md` 通过了过滤，被当作需要校验路径的镜像级文档。
3. `parse_image_prefix("README.md")` 对单段路径返回 `("", "")`（`if len(contents) == 1: return "", ""`），于是前缀为空。
4. `_check_all_file_paths()` 计算 `correct_path = "{0}/README.md".format("")` 得到 `/README.md`，再执行 `os.path.exists("/README.md")` 必然为 False，从而返回 `[Path Error] The expected path should be /README.md`，最终 `fail_count > 0`，`check_code()` 报错并标记构建失败。

该判定完全基于**文件名**而非文件内容：只要变更集中出现根目录的 `README.md`（`README.en.md` 同样会被识别为 `README` 类型），校验就必然失败。因此对 `README.md` 做任何内容层面的修改都无法绕过该判定。

正确的修复应位于 CI 侧的外部工具 `eulerpublisher/update/container/app/format.py`（例如：当 `parse_image_prefix()` 返回空前缀时跳过根目录文档，或从 `DOC_FILES_PATH_FORMAT` 中排除根级 README），但该文件不属于本仓库内容，也不在原始 PR 的 `changed_files`（仅 `README.md`）范围内。按照最小化修复原则，不对与失败无直接关系的文件做改动。

已从上游 `openeuler/eulerpublisher` 仓库（master 分支）获取 `update/container/app/update.py` 与 `update/container/app/format.py` 实际源码，并在本地用 Python 复现了上述判定流程：`file_type == "README"`、`correct_path == "/README.md"`、`os.path.exists("/README.md") == False`，与 CI 日志完全一致，确认根因为工具缺陷而非 README 内容问题。

## 潜在风险
无。本次未做任何代码修改，不会影响仓库其他功能。建议该 PR 的失败按 CI 工具误报处理，由 CI/EulerPublisher 维护方修复根目录 README 的校验规则后再重跑门禁。
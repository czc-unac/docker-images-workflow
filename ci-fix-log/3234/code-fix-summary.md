# 修复摘要

## 修复的问题
本次 CI 失败为 EulerPublisher 校验工具对仓库根目录 `README.md` 的路径归一化缺陷（infra-error），与本 PR 的文档内容无关；无法通过修改 `README.md` 内容修复，故不做代码修改。

## 修改的文件
- 无（未修改任何源码文件）

## 修复逻辑
分析报告将失败类型标为 `lint-error`，并给出两个修复方向（方向 1：把文档放到 CI 允许的目录；方向 2：判定为校验工具误报）。经查阅上游 `eulerpublisher` 源码后确认属于**方向 2（工具缺陷 / infra-error）**：

1. CI 校验入口为 `update/container/app/update.py:271` 的 `check_code()`，它调用 `format.check_report(self.change_files)`；`change_files` 为 PR 变更文件，本次仅 `README.md`。
2. `update/container/app/format.py` 中 `DOC_FILES_PATH_FORMAT` 含有键 `"README": "{0}/README.md"`（设计意图是校验**镜像目录级**的 `{image-prefix}/README.md`）。
3. `check_report()`（format.py:181-193）用 `change_file.split("/")[-1].split(".")[0]` 得到 `README`，命中 `DOC_FILES_PATH_FORMAT`，因此**仓库根目录的全局 README 也会被当作镜像 README 校验**。
4. `parse_image_prefix("README.md")`（format.py:119-125）对单段路径返回空前缀 `("", "")`，于是 `_check_all_file_paths()`（format.py:248-252）计算 `correct_path = "{0}/README.md".format("", "README.md")` = `/README.md`，`os.path.exists("/README.md")` 恒为 False → 报 `[Path Error] The expected path should be /README.md`。

即：**任何触及仓库根 `README.md` 的 PR 都会被该路径检查误判失败**，而期望路径 `/README.md` 在任何仓库中都不存在。该检查仅依赖文件名，与 `README.md` 的正文内容无关，因此在“只允许修改 `README.md`、不允许新增/移动文件”的约束下，不存在能通过该校验的内容改动；问题必须由 `eulerpublisher` 修复（例如在 `_check_all_file_paths`/`check_report` 中对空前缀的根级文件跳过检查，或用 `os.path.join` 而非字符串 `.format` 拼接）。

补充证据（与本地代码库核对一致）：
- 本地 `git diff` 显示本 PR 仅 `README.md` +12 行，与 CI `Difference: ["README.md"]` 完全对应。
- 仓库历史上根目录 `README.md` 已被多次正常修改并合入（如 master 上 `5c99b50d6 docs: update available base image tags in README`），说明修改根 README 本身是被允许的，进一步证明本次为工具误报。
- 知识库模式 11 的历史 README 路径校验案例（PR #2512 的 `.claude/.../README.md`）同源，均因该 README 路径归一化逻辑产生误报。

依据 Agent 约定“分析报告指出 infra-error 时说明无需代码修改，不要强行改代码”，本次不产生提交（workflow 将按 `no_changes` 处理）。

## 潜在风险
无。未修改任何文件，不影响仓库其他功能。真正需要修复的位置在外部工具 `eulerpublisher/update/container/app/format.py`（`DOC_FILES_PATH_FORMAT` 的 `README` 处理 / 空前缀路径拼接），需由该工具维护方处理。
# 修复摘要

## 修复的问题
未修改任何源码文件：经独立验证，CI 失败根因是上游 EulerPublisher 工具对仓库根目录 `README.md` 的路径校验缺陷（infra/tooling 缺陷），不属于本 PR 的代码或文档问题，且无法在允许修改的文件（仅 `README.md`）范围内修复。

## 修改的文件
- 无（未对源码仓库做任何改动）

## 修复逻辑
### 根因验证（已拉取上游实际源码）
通过 GitHub 拉取了 `openeuler-mirror/eulerpublisher@master` 的
`update/container/app/update.py` 与 `update/container/app/format.py`，逐行确认了失败链路：

1. `ContainerVerification.get_change_files()` 取得本次 PR 变更集 `["README.md"]`（即日志中的 `Difference`）。
2. `check_code()` 调用 `format.check_report(["README.md"])`。
3. `check_report()` 中 `file_type = "README".split(".")[0] == "README"`，命中 `DOC_FILES_PATH_FORMAT`，进入 `_check_all_file_paths("README.md")`。
4. `parse_image_prefix("README.md")`：`contents = ["README.md"]`，`len(contents) == 1`，直接 `return "", ""`，即 **prefix 为空字符串**。
5. `_check_all_file_paths()`：
   `correct_path = DOC_FILES_PATH_FORMAT["README"].format("", "README.md")`
   `DOC_FILES_PATH_FORMAT["README"] = "{0}/README.md"`
   代空 prefix 后结果为字符串 **`"/README.md"`**（被解释为文件系统根目录的绝对路径）。
6. `os.path.exists("/README.md")` 在 CI 工作目录（`/tmp/.../ci/container/<repo>`）下为 **False**，于是返回
   `[Path Error] The expected path should be /README.md`，`fail_count` 加一，最终 `update.py[line:273]` 报错并以 FAILURE 结束。

结论：**该报错与 `README.md` 的内容完全无关**。失败源于工具在“根目录单段路径”场景下
`parse_image_prefix` 返回空 prefix，导致期望路径被拼成绝对路径 `/README.md`。即使修改
`README.md` 的任意内容，只要变更文件仍是仓库根的 `README.md`，该检查依然会失败。

### 与 CI 分析报告的一致性
- 报告的“方向 2”（核对 `update.py` 路径归一化/比对逻辑，确认为工具缺陷后按 CI 侧问题处理）**已被源码验证成立**。
- 报告明确指出“置信度为低、失败信息自相矛盾，Code Fixer 不得据此直接修改 `README.md` 内容”。本次验证进一步证实该结论。
- 真正可行的修复应落在上游 `eulerpublisher/update/container/app/format.py`
  （例如：当 prefix 为空时按相对路径 `README.md` 校验，或对根目录文档文件跳过镜像发布规范校验），
  但该文件不在本 PR 的 `changed_files`（仅 `README.md`）内，按约束不可修改。

## 潜在风险
无（本次未做任何代码改动）。
需注意：若后续仍需消除该门禁失败，应由 CI/上游工具侧修复 `format.py`，或为该纯文档 PR 增加豁免规则；
在工具缺陷修复前，本 PR 的该检查项无法通过修改仓库文档来规避。
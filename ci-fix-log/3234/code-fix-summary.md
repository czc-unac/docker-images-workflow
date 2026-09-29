# 修复摘要

## 修复的问题
未做代码修改：本次失败并非本仓库代码缺陷，而是上游 CI 校验工具 `eulerpublisher`
的路径校验逻辑存在 bug——任何以仓库根目录 `README.md` 作为变更文件的 PR 都必然
报 `[Path Error] The expected path should be /README.md`。该问题无法通过修改
`README.md` 内容解决，属于 CI 工具/基础设施问题（infra/tool bug）。

## 修改的文件
- 无。分析报告允许修改的唯一文件为 `README.md`，但对 `README.md` 的任何内容改动
  都无法改变校验结果（校验基于文件名/路径而非内容），因此未提交任何改动。

## 修复逻辑
根因已通过本地复现确认（并非推断）：

1. appstore 发布规范校验位于 `eulerpublisher/update/container/app/format.py`
   的 `check_report()` / `_check_all_file_paths()`。它会遍历 PR 变更文件，对
   文件名（去扩展名）命中 `DOC_FILES_PATH_FORMAT` 的文件做路径合规检查。
2. `README.md` 的去扩展名是 `README`，命中 `DOC_FILES_PATH_FORMAT`，因此被纳入
   校验。
3. `parse_image_prefix("README.md")` 中因 `file.split("/")` 长度为 1，函数提前
   `return "", ""`，得到空前缀 `prefix = ""`。
4. 随后 `_check_all_file_paths()` 计算期望路径：
   `DOC_FILES_PATH_FORMAT["README"].format("", "README.md")`，而
   `DOC_FILES_PATH_FORMAT["README"] = "{0}/README.md"`，于是 `correct_path`
   变成绝对路径 `/README.md`。
5. `os.path.exists("/README.md")` 在文件系统根目录下恒为 `False`，于是返回
   `False, "[Path Error] The expected path should be /README.md"`，计入
   `fail_count`，导致门禁失败。

本地已用与 CI 同款的 `format.py` 复现：
```
parse_image_prefix('README.md') = ('', '')
correct_path = '/README.md'
exists('/README.md') = False
+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+
```

结论与证据：
- 该 bug 只与“变更文件是否为仓库根 `README.md`”有关，与 `README.md` 的正文内容
  完全无关，因此在本仓库内对 `README.md` 做任何增删改都无法使校验通过。
- 同一逻辑下，根目录 `README.en.md`（去扩展名同样是 `README`）也会被算成
  `/README.md`，同样必然失败。
- 正确的修复应在 CI 侧工具：当 `prefix` 为空时应直接使用原变更文件路径本身
  （`change_file`）作为期望路径，或在路径校验中为仓库根层级的 `README*` 增加白名单；
  这属于 `eulerpublisher` 仓库的改动，不在本 PR 的 `changed_files`（仅 `README.md`）
  范围内，也非本仓库源码问题。

对照分析报告的两个方向：
- 方向 1（调整文档存放位置）：本次任务约束只允许修改 `README.md`，不允许新增/移动
  文件，故不可行；且校验是针对根文件名的工具缺陷，换位置也无法在约束内完成。
- 方向 2（CI 工具误判）：已核实成立，即本次失败系 `format.py` 对根层 README 的路径
  计算缺陷所致，不应改动本仓库文件。

## 潜在风险
无（未产生任何仓库改动）。建议由 CI/门禁维护方修复 `eulerpublisher` 的
`update/container/app/format.py`：在 `prefix` 为空时使用原文件路径、或对仓库根
`README.md`/`README.en.md` 加入路径白名单，之后本 PR 即可正常通过门禁。
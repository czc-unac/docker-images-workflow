# 修复摘要

## 修复的问题
无需修改本仓库代码：本次 CI 失败是 appstore 发布规范预检对「纯文档改动（根目录 `README.md`）」的误判，根因在外部工具 EulerPublisher 的路径校验逻辑，不在本 PR 的源码内。

## 修改的文件
- （无）未修改 `README.md` 或其他任何文件。

## 修复逻辑
已核对 CI 校验源码（从上游 `openeuler-mirror/eulerpublisher@master` 拉取 `update/container/app/format.py` 与同目录 `update.py` 验证），根因明确为路径校验工具对根目录文档文件的处理缺陷：

1. `update.py:check_code()` 调用 `format.check_report(change_files)`，PR 变更清单为 `["README.md"]`（日志 `Difference: ["README.md"]`）。
2. `check_report()` 中：`file_type = change_file.split("/")[-1].split(".")[0]`，对根目录 `README.md` 得到 `"README"`；该值恰好在全局字典 `DOC_FILES_PATH_FORMAT` 的键中，因此 **不会被 `continue` 跳过**，被当作「镜像文档文件」纳入路径校验。
3. `parse_image_prefix("README.md")` 因 `len(contents) == 1`（根目录文件无上级目录）直接返回 `("", "")`，即 `prefix` 为空串。
4. `_check_all_file_paths()` 用空 `prefix` 拼出期望路径：`DOC_FILES_PATH_FORMAT["README"].format("", "README.md")` = `"/README.md"`，随后执行 `os.path.exists("/README.md")`——这是**文件系统根目录的绝对路径**，并非仓库内的 `README.md`，必然不存在，于是报 `[Path Error] The expected path should be /README.md` 并使预检 FAILURE。

也就是说，`README.md` 本就位于仓库根目录、路径完全正确，失败源于校验器把空前缀拼成了以 `/` 开头的绝对路径，且未跳过「不属于任何镜像目录」的文档文件。因此：

- 本 PR 的文档新增内容（通过 Issue 自动化新增镜像指南）本身没有问题，不应删除或改写；在 `pr.changed_files = ["README.md"]` 范围内也不存在任何能让该校验通过的改动（README 的内容与文件路径均不影响 `parse_image_prefix` 的判定）。
- 正确的修复应在上游 EulerPublisher 侧，属于本仓库之外、且不在允许修改的文件列表中：
  - 在 `check_report()` 中，当 `parse_image_prefix()` 返回空 `prefix`（即文件不在任何镜像目录内）时跳过该校验；或
  - 对根层级文档文件单独处理，避免 `"{0}/README.md".format("")` 产生绝对路径。

按「不修改 CI 配置绕过检查、不触碰 `pr.changed_files` 之外文件」的约束，本次不做代码改动，交由此修复流程按「无需修改」处理。

## 潜在风险
无。未对仓库做任何修改，不存在引入回归的风险。

补充说明：该失败与「修改正则 patch 外部源文件」无关，无需正则验证项。若后续上游 EulerPublisher 修正 `check_report()` 对根目录文档文件的误判，本 PR 可直接重新触发 CI 通过。
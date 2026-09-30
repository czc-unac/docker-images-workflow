# 修复摘要

## 修复的问题
本次 CI 失败不是本仓库（openeuler-docker-images）的源码缺陷，而是上游 CI 工具 eulerpublisher 的 appstore 发布规范预检对“仓库根目录文档文件”的路径判定缺陷：它把根目录 `README.md` 误当作“镜像目录内的 README”来计算期望路径，规范化出一个绝对路径 `/README.md`，该路径在 CI 容器中永远不存在，于是无条件报 `[Path Error] The expected path should be /README.md`。任何只修改 `README.md` 内容的改动都无法满足该校验，因此不做源码修改（强行改 README 内容既不能通过门禁，也会破坏原 PR 的文档贡献）。

## 修改的文件
- 无（未修改任何文件）

## 修复逻辑
1. 已在当前仓库核对失败来源：`README.md` 是 PR 唯一变更文件，CI 调用链为 `update/container/app/update.py:270 check_code()` → `format.check_report(self.change_files)` → `format._check_all_file_paths()`。
2. `format.check_report()`（format.py:181-193）对每个变更文件取 `file_type = basename.split(".")[0]`，`"README"` 命中全局表 `DOC_FILES_PATH_FORMAT`，因此根目录 README 不会被跳过。
3. `format.parse_image_prefix("README.md")`（format.py:119-125）因路径不含 `/`（`len(contents) == 1`）直接返回 `("", "")`，即 `prefix` 为空串。
4. `format._check_all_file_paths()`（format.py:247-252）计算
   `correct_path = "{0}/README.md".format("", "README.md") == "/README.md"`，
   再执行 `os.path.exists("/README.md")`。该路径是**文件系统绝对根路径**（不是仓库相对路径），在 CI 容器中必然不存在，于是返回
   `[Path Error] The expected path should be /README.md`，`fail_count` 累加，`update.py:273` 打印错误并以 FAILURE 退出。
5. 关键结论：该判定只依赖 **PR 变更文件列表 + 文件名的 basename**，与 README.md 的**内容完全无关**。只要“变更文件集合”里含有仓库根目录、且 `basename.split(".")[0]` 命中 `DOC_FILES_PATH_FORMAT`（`README`/`meta`/`doc`/`picture`/`logo`/`image-info`）的文件，就必然失败。因此：
   - 本仓库内不存在可修复该失败的源码改动（eulerpublisher 为外部依赖，不在 `pr.changed_files`，也不在源码仓库中）；
   - 对 `README.md` 做任何内容编辑（保留、删除新增段落、加版权头等）都不会改变 `correct_path` 的计算结果，CI 仍会以同样的 `[Path Error]` 失败；
   - 唯一能通过的“内容级”操作是让该文件不再是变更文件（如删除根 README），这属于破坏性操作且违背最小化原则，不采用。
6. 正确的修复应在上游 eulerpublisher 工具侧完成。已从上游 `openeuler/eulerpublisher`（master）拉取实际源文件 `update/container/app/update.py` 与 `update/container/app/format.py` 进行核对，二者行号与 CI 日志完全一致（update.py:356 `Difference`、update.py:273 报错），确认定位无误。建议上游修改（供维护方参考，不在本仓库实施）：
   ```python
   # format.check_report() 中，跳过仓库根目录等无法归属到镜像目录的文件
   _, prefix = parse_image_prefix(change_file)
   if not prefix:
       continue
   ```
   或在 `_check_all_file_paths()` 中当 `prefix == ""` 时直接返回成功。
7. 因此本阶段的处理为「无需代码修改（no-changes）」：该校验对 docs-only、且改动位于仓库根目录的 PR 属于误报，需由仓库维护方确认根目录 README 变更豁免该校验，或由上游修复 `format.py`。这与 CI 分析报告“方向 1”中的第二个选项（由仓库维护方确认豁免）一致。

## 潜在风险
无。未对源码仓库做任何修改，不会影响任何镜像构建、发布或既有功能。遗留影响仅为：在 eulerpublisher 修复该路径判定之前，任何改动仓库根目录 `README.md` 等文档类文件的 PR 都会被 appstore 规范预检误判失败，需要维护方豁免或上游工具修复。
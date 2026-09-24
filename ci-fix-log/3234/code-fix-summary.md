# 修复摘要

## 修复的问题
CI 失败（appstore 发布规范路径校验：`README.md` → `[Path Error] The expected path should be /README.md`）经上游源码级核实，属 **infra-error**——EulerPublisher 的 appstore 规范校验工具 (`format.check_report`) 对仓库根目录文档存在路径归一化缺陷。PR #3234 为纯文档变更，内容本身无问题，本次未做任何代码修改（也不应强行修改 README.md 内容来迎合该校验）。

## 修改的文件
- 无。`README.md` 保持 PR 原有内容，未做改动。

## 修复逻辑
不修改代码的依据来自对校验工具实际源码的分析与复现：

1. 校验入口：`update/container/app/update.py:270` 调用 `format.check_report(self.change_files)`；失败在 `update.py:272`（`fail_count` 非 0）触发。
2. 关键逻辑在 `update/container/app/format.py:163` 的 `check_report`：
   - `format.py:184` 用 `change_file.split("/")[-1].split(".")[0]` 取文件类型。对根目录 `README.md` 该值为 `"README"`。
   - `format.py:185` 仅当 `file_type not in DOC_FILES_PATH_FORMAT` 时才 `continue`，而 `DOC_FILES_PATH_FORMAT` 恰好包含键 `"README"`（`format.py:15`），因此根目录 README 没有被跳过，反而进入镜像文档校验流程。
   - `format.py:119` 的 `parse_image_prefix("README.md")`：由于路径只有一段（`len(contents) == 1`），在 `format.py:124` 直接返回 `("", "")`，`prefix` 为空。
   - `format.py:248` 用 `DOC_FILES_PATH_FORMAT["README"].format("")` 得到期望路径 `"/README.md"`，再在 `format.py:251` 用 `os.path.exists("/README.md")` 判断（工作目录为 clone 出来的仓库，根为仓库根，故 `"/README.md"` 实际是文件系统绝对根路径，必然不存在），于是 `format.py:252` 返回 `[Path Error] The expected path should be /README.md`。
3. 本地按上述逻辑复现（对 `README.md`）：
   `README.md | file_type=README | prefix='' | expected=/README.md | exists=False` —— 与 CI 日志完全一致。

结论：**任何对仓库根目录 `README.md` 的修改都会触发同一误报**，与 PR 新增的 12 行文档内容无关。分析报告的"方向 1"（把内容搬到规范约定的镜像文档目录）在当前约束下不可行——本仓库根目录文档的合法替代位置需要新建文件/修改 `pr.changed_files` 之外的文件，均被禁止；且根因是工具把"非镜像目录下的 README"误当作镜像 README 校验。按流程约定，此类 `infra-error` 无需（也不应）改动 PR 代码，应由 CI/工具维护方修复 `format.check_report`（例如对不含 `/` 的根目录文件直接跳过，或 `parse_image_prefix` 返回空前缀时跳过校验）。

### 工具侧建议修复（供转交 CI/工具维护方，非本次改动范围）
- 在 `format.py:185` 的跳过条件中增加"路径不含 `/`（即非镜像目录内文件）则跳过"，或
- 在 `format.py:_check_all_file_paths` 中当 `prefix` 为空时直接返回通过。

## 潜在风险
无。本次未对源码库做任何修改，不会引入回归；唯一结果是该校验在工具侧修复前，根目录 `README.md` 的 CI 检查仍会误报。
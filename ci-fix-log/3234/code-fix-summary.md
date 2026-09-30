# 修复摘要

## 修复的问题
未做代码修复：本次 CI 失败是 eulerpublisher appstore 路径预检对**仓库根目录 `README.md`** 的误报（工具侧缺陷），非 PR 内容问题；在允许修改的文件范围内不存在能使该检查通过的合法改动。

## 修改的文件
- 无（未修改任何文件；`README.md` 保持 PR 原样）

## 修复逻辑

### 1. 已核对 CI 校验逻辑（按分析报告第 4 点要求）
从上游 `openeuler-mirror/eulerpublisher`（master）拉取并核对了实际校验代码：
- `update/container/app/update.py` → `ContainerVerification.check_code()` 调用 `format.check_report(self.change_files)`
- `update/container/app/format.py` → `check_report()` / `_check_all_file_paths()` / `parse_image_prefix()`

对本次唯一变更文件 `README.md` 逐步代入校验逻辑，失败链路如下：

1. `check_report(["README.md"])` 中 `file_type = "README"`，命中全局字典 `DOC_FILES_PATH_FORMAT`，于是进入路径校验；
2. `parse_image_prefix("README.md")`：函数开头 `if len(file.split("/")) == 1: return "", ""`，**根目录文件返回空前缀**；
3. `_check_all_file_paths("README.md")` 计算 `correct_path = DOC_FILES_PATH_FORMAT["README"].format("", "README.md")` = `"/README.md"`；
4. `os.path.exists("/README.md")` 为 False（`"" + "/README.md"` 变成了**文件系统绝对路径** `/README.md`，而非仓库相对路径）；
5. 返回 `[Path Error] The expected path should be /README.md`，`fail_count += 1`，触发 `update.py:273` 的 `logger.error(...)` 并 `exit(1)`，job FAILURE。

即：**只要 PR 变更集合里出现仓库根目录的 `README.md`，该校验必然报路径错误**，与 README 的具体文字内容无关。已在本地用 Python 完整复现该判定（`prefix=''`、`correct_path='/README.md'`、`os.path.exists=False`）。

### 2. 为什么不做“方向 1”的改动
- 校验结果只取决于**文件路径**（`parse_image_prefix` 对无 `/` 的文件固定返回 `""`），与 README 正文内容完全无关。因此**无论怎样编辑 `README.md` 的内容，`correct_path` 都恒为 `/README.md`，检查必然失败**。
- 分析报告“方向 1”要求把文档移动到校验器认可的路径（如 `{prefix}/README.md`、`{prefix}/doc/...`）。这需要**新增/移动文件**（例如删除根 `README.md` 并在其他目录新建文件），而本任务明确约束：**只允许修改 `pr.changed_files` 中已存在的 `README.md`，禁止创建任何新文件**。故方向 1 在本任务的允许范围内不可执行。
- 根目录 `README.md` 是仓库自身的说明文档，并非镜像“最小目录单元”内的 README，本不应被 appstore 发布规范校验。PR #3234 仅新增一段“通过 Issue 自动新增应用镜像”的使用指引，是合法文档改动。

### 3. 结论
根因位于 CI 预检工具（`format.py`）对**无路径分隔符的根级文档**缺少跳过逻辑，属于工具侧规则缺陷（分析报告“方向 2”），无法且不应通过修改 PR 的 `README.md` 来消除。建议由维护者在 `format.py` 中修复，例如：`check_report` 在 `file_type in DOC_FILES_PATH_FORMAT` 后增加 `prefix` 非空判断（空前缀即仓库根级文档，跳过），或将 `parse_image_prefix` 对单段路径返回空前缀的情况排除。此修复不在本仓库 `pr.changed_files` 范围内，故本任务**不做任何代码改动**。

## 潜在风险
无。未修改任何源码文件，不引入回归。
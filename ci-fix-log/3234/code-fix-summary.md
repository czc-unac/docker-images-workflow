# 修复摘要

## 修复的问题
经定位，本次失败为 CI 上游预检工具 `eulerpublisher` 的路径计算缺陷（对仓库根目录的 `README.md` 错误地要求其位于文件系统绝对路径 `/README.md`），并非 `README.md` 内容问题；因该工具不在本 PR 允许修改的文件范围内，且对 `README.md` 的任何内容改动都无法改变被校验的文件路径，故未对源码做强行修改。

## 修改的文件
- 无。`README.md` 未做任何修改（分析报告指向的检查逻辑缺陷位于 CI 工具 `eulerpublisher`，不在 `pr.changed_files` 允许范围内，且无内容层面的修复手段）。

## 修复逻辑

### 根因确认（已通过阅读上游源码复现）
CI 在 `update/container/app/build.sh` 中 `git clone https://gitee.com/openeuler/eulerpublisher.git` 后执行 `python3 update/container/app/update.py ... -op check`，其中的 `check_code()` 调用 `format.check_report(change_files)`：

1. `format.py:163 check_report()` 遍历变更文件，用 `change_file.split("/")[-1].split(".")[0]` 取类型。对 `README.md` 得到 `README`，而 `README` 恰好在 `DOC_FILES_PATH_FORMAT`（`format.py:14-22`）中，因此**根目录 README 不会被跳过**。
2. `format.py:119 parse_image_prefix("README.md")` 中 `contents = ["README.md"]`，`len(contents) == 1`，直接返回 `(prefix, "") = ("", "")`。
3. `format.py:234 _check_all_file_paths("README.md")` 计算
   `correct_path = DOC_FILES_PATH_FORMAT["README"].format("", "README.md") = "/README.md"`，
   随后 `os.path.exists("/README.md")` 检查的是**文件系统的根**，而非仓库工作目录，必然不存在，于是返回
   `[Path Error] The expected path should be /README.md`，`fail_count` 加一。
4. `update.py:272-276` 因 `fail_count` 非 0 记录 ERROR 并以非零码退出，构建标记为 FAILURE。

本地对上述逻辑的复现结果：
```
'README.md'            type='README' prefix='' expected_path='/README.md' exists=False
```
与 CI 日志完全一致（`Difference: ["README.md"]`、`The expected path should be /README.md`）。这说明失败由**根目录文档文件触碰了“镜像文档路径校验”**导致，属于预检工具对仓库根级文档的误判/缺陷（`format.py` 未过滤无目录前缀的文件，且用空前缀拼接出了以 `/` 开头的绝对路径）。

### 为何不能在 `README.md` 内修复
- 被校验对象是“变更文件的路径”，由 git diff / 平台 API 给出，固定为 `README.md`，与文件内容无关。对 `README.md` 做任何增删改（包括回退新增文档）都不会改变 `_check_all_file_paths` 计算出的 `/README.md`。
- 唯一能让该校验通过的方式是修改 CI 工具逻辑（`format.check_report` 跳过无目录前缀的变更文件，或 `_check_all_file_paths` 在 `prefix` 为空时按仓库根 `.` 处理，例如期望路径取 `./README.md`）。该文件属于上游 `eulerpublisher` 仓库（CI 运行时从 gitee 动态克隆安装），既不在本仓库内，也不在 `pr.changed_files` 中，受约束不可修改。

### 处置
按“infra / 工具缺陷不强行改代码”的原则，本次不产生源码改动。建议由流水线维护方在上游 `eulerpublisher` 修复：
- 在 `format.check_report()`（`update/container/app/format.py:181-193`）中，对不含 `/` 的变更文件（即仓库根级文件，`parse_image_prefix` 返回空 prefix）直接 `continue`，不纳入 appstore 规范路径校验；或
- 在 `_check_all_file_paths()`（`format.py:234-253`）中，当 `prefix == ""` 时使用仓库根 `.` 而非空串拼接，避免产生 `/README.md` 这类绝对路径。

## 潜在风险
- 无源码改动，不引入任何回归风险。
- 说明：本修复未使该 PR 的 CI 转绿——因为导致失败的检查逻辑不在允许改动的文件范围内。在流水线维护方修正 `eulerpublisher` 之前，任何仅修改仓库根 `README.md` / `README.en.md` 的纯文档 PR 都会复现同一失败（与 [docs/ci-failure-patterns.md 模式11](../../../docs/ci-failure-patterns.md) 中 `.claude/*/README.md` 的 appstore 路径校验失败同源）。若必须让当前 PR 通过门禁，需由维护方调整预检范围或放行根级文档变更。
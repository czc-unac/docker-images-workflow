# 修复摘要

## 修复的问题
本次 CI 失败为 EulerPublisher（上游 CI 工具）发布规范预检的**误报**，不是本仓库代码问题；未对源码做任何修改。

## 修改的文件
- 无（未修改任何文件）

## 修复逻辑
已从上游仓库获取并核对实际源文件，确认根因位于 EulerPublisher 的 `update/container/app/format.py`：

1. `check_report()` 遍历变更文件时，用 `change_file.split("/")[-1].split(".")[0]` 取文件类型。对根目录 `README.md`，得到 `README`，而 `README` 正是 `DOC_FILES_PATH_FORMAT` 的合法键，因此**没有被跳过**。
2. 随后调用 `parse_image_prefix("README.md")`。该函数中 `contents = ["README.md"]`，`len(contents) == 1` 时直接 `return "", ""`，于是前缀 `prefix = ""`。
3. `_check_all_file_paths()` 用 `DOC_FILES_PATH_FORMAT["README"].format(prefix, ...)` 计算正确路径，得 `"" + "/README.md" = "/README.md"`。
4. `os.path.exists("/README.md")` 在文件系统根目录下为 False，于是报出 `[Path Error] The expected path should be /README.md`，预检 `fail_count` 增加，`check_code()` 返回 1，流水线失败。

这与 CI 日志完全一致：
```
Difference: [ "README.md" ]
| README.md | [Path Error] The expected path should be /README.md | FAILURE |
```

**已从上游 master 分支获取 `update/container/app/update.py` 与 `update/container/app/format.py` 验证**，并用 Python 复现了上述路径计算（`prefix=''` → `correct_path='/README.md'` → `exists=False`）。

### 为什么无需在本仓库改代码
- 该判定只依赖**文件路径**，与 `README.md` 的内容无关。只要 PR 修改仓库根目录的 `README.md`，无论内容如何，都会被同一逻辑判为 `/README.md` 不存在而失败。因此在本仓库内无法通过修改 `README.md` 内容规避。
- 分析报告“方向 1”（把内容移到其他文档位置）需要新增文件，而任务约束明确**禁止创建任何新文件**，且只允许修改 `pr.changed_files` 内的 `README.md`，方案不可行；同时把内容挪走也不属于对失败根因的修复。
- 真正应有的修复在上游工具 `eulerpublisher/update/container/app/format.py`：当 `parse_image_prefix()` 返回空前缀（即文件不在任何镜像最小目录内）时，应跳过路径校验；或仅在前缀非空时才把 `README` 等文档类型计入预检。这属于 CI 工具/基础设施问题，按任务约定 Code Fixer 不强行改代码，建议联系 CI 运维（Infra SIG / EulerPublisher 维护者）修正该预检逻辑。

## 潜在风险
无。本次未做任何代码改动，不影响仓库任意功能。后续 PR 若仍需修改根目录 `README.md`，在该上游预检 bug 修复前仍会复现同类失败。
# 修复摘要

## 修复的问题
经核实，本次 CI 失败是 EulerPublisher 发布规范预检工具自身的路径拼接缺陷（infra/tool bug），不是本仓库 `README.md` 内容的问题，因此无需对源码仓库做任何代码修改。

## 修改的文件
- 无（未修改任何文件；本 PR 唯一涉及文件 `README.md` 无任何可使其通过预检的内容改动）

## 修复逻辑
通过拉取上游预检源码 `https://gitee.com/openeuler/eulerpublisher/raw/master/update/container/app/format.py` 逐行定位，确认失败链路如下：

1. `update/container/app/update.py` 的 `check_code()` 调用 `format.check_report(self.change_files)` 做 appstore 发布规范预检。
2. `format.py` 的 `check_report()` 第 1 步遍历变更文件：对 `README.md`，`file_type = "README"`，该类型确实存在于 `DOC_FILES_PATH_FORMAT = {"README": "{0}/README.md", ...}`，因此**不会被跳过**，被纳入路径校验。
3. `parse_image_prefix("README.md")` 中 `contents = ["README.md"]`，`len(contents) == 1`，函数直接返回 `("", "")`，即 `prefix = ""`。
4. `_check_all_file_paths("README.md")` 计算 `correct_path = DOC_FILES_PATH_FORMAT["README"].format("", "README.md")`，字符串模板 `"{0}/README.md"` 在 `{0}` 为空时产生**前导斜杠**，得到 `/README.md`。
5. `os.path.exists("/README.md")` 恒为 False（指向文件系统根目录），于是返回 `(False, "[Path Error] The expected path should be /README.md")`，`fail_count` 加一，`check_code()` 打印 `There are some specification errors...` 并返回非 0，构建被标记为失败。

我用相同逻辑在 Python 中复现，确认 `"{0}/README.md".format("") == "/README.md"` 且 `os.path.exists("/README.md")` 为 False，与分析报告的失败表项完全一致。

结论：`DOC_FILES_PATH_FORMAT` 仅用于**镜像最小目录单元下的文档**（如 `AI/opea/README.md`），而仓库**根目录的 `README.md`** 被错误地套用了同一模板；由于 `prefix` 为空，期望路径变成绝对路径 `/README.md`，从而对**任何修改根目录 README 的 PR（不仅是文档类 PR）都必然误报**。这是 `format.py` 的空前缀处理缺陷，修复点应位于 `eulerpublisher` 工具侧（例如当 `prefix` 为空时令 `correct_path = contents[-1]`，或在 `check_report` 中跳过仓库根级文档），而**不在本仓库**。本 PR 仅新增了 12 行“通过 Issue 自动新增应用镜像”的说明文档，未触碰任何镜像目录、Dockerfile、`meta.yml` 或 `image-list.yml`，故失败由工具在纯文档变更上误触发，而非文档内容本身。

对应分析报告的“修复方向·方向 2（置信度低）”：现已由上游源码证实预检的路径归一化确有缺陷，属流水线/工具侧问题（infra）。分析报告“方向 1”中可落地的仓库侧做法（迁移到规范路径或拆分为独立新文档文件）均与本次任务约束冲突——本任务明确**只允许修改 `README.md` 且禁止创建任何新文件**；而 `README.md` 作为文件名，其 `file_type` 恒为 `"README"`、`prefix` 恒为空，无论内容如何修改都无法改变上述判定，故不存在满足约束的仓库侧修复。

## 潜在风险
无代码改动，不引入任何回归。需注意：在 `eulerpublisher` 修复该缺陷前，任何以修改仓库根目录 `README.md` 为唯一/主要内容的 PR 都会重复触发同一误报，建议将本结论反馈给流水线维护方（`eulerpublisher` 仓库）修复 `update/container/app/format.py` 中的空前缀处理。
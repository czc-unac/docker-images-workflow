# 修复摘要

## 修复的问题
无需代码修改：该失败并非由 PR 变更内容引起，而是 CI 校验器（EulerPublisher `update/container/app/format.py`）对“根目录单段路径文件”的路径归一化缺陷导致，属工具/基础设施侧问题。

## 修改的文件
（无）

## 修复逻辑

按分析报告要求，先确认了第 1、2 点（校验器期望路径生成规则、变更文件纳入范围），通过拉取上游 EulerPublisher 实际源码（`update/container/app/format.py` 与 `update/container/app/update.py`）取证，结论如下：

1. **失败入口与范围**：`update.py:356` 打印的 `Difference` 来自 PR 变更文件清单，`update.py:270` 将其交由 `format.check_report()`；`format.py:181-193` 会对每个变更文件做路径校验，只要文件名（去扩展名）命中 `DOC_FILES_PATH_FORMAT`（含 `README`）就参与校验。因此任意 `README.md` 变更（无论内容）都会被纳入 appstore 预检。

2. **期望路径生成缺陷**：`format.py:119-125` 的 `parse_image_prefix()` 对不含 `/` 的单段路径（根目录文件）直接返回 `("", "")`；随后 `format.py:247-252` 用 `DOC_FILES_PATH_FORMAT["README"].format("", "README.md")` 得到 `"/README.md"`。该结果被当作**绝对路径**交给 `os.path.exists()`，而仓库里只存在相对路径 `README.md`，根目录 `/README.md` 不存在，于是必然返回 `[Path Error] The expected path should be /README.md`。

3. **本 PR 关联**：PR 仅修改根目录 `README.md`（`git diff --name-only origin/master...HEAD` 唯一文件），恰好触发上述缺陷。历史上该仓库根目录 `README.md` 被多次正常修改（如 `update os order`、`Add readme.en changes`），说明根目录文档变更在流程上是合法的。

4. **为何不做代码修改**：
   - 该校验是**纯路径校验**，不读取文件内容。任何对根目录 `README.md` 的内容改动都无法改变校验器计算出的 `prefix=""` 与 `correct_path="/README.md"`；`README.en.md`、迁移内容到其他文件、新增文件等路径均被本任务约束（只允许改 `README.md`、禁止新增文件）排除，且同样命中该缺陷。
   - 若“消除变更”以通过校验，则等于删除本 PR 的唯一目的（新增自动化新增镜像指南），属于为绕过有缺陷的门禁而丢弃合法变更，不应由代码修复承担。
   - 因此根因在 CI 工具侧：`parse_image_prefix` / `_check_all_file_paths` 未对根目录单段路径做归一化（应跳过根目录文档文件，或将 `""` 前缀正确拼为 `README.md` 而非 `/README.md`）。按流程这属于 `infra-error`/工具缺陷，应由 CI 维护方修复 `format.py`。

综上，本任务按“无需代码修改”处理，未改动 `README.md` 或任何其他文件。

## 潜在风险
无（未修改任何源文件）。若上游 CI 维护方不修复 `format.py` 的根路径归一化逻辑，则本 PR 及后续任何根目录 `README.md`/`README.en.md` 变更都会持续因该门禁失败；建议将本结论反馈给 CI/EulerPublisher 维护方。
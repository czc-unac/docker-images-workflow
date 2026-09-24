# 修复摘要

## 修复的问题
经复现确认，本次 CI 失败是 EulerPublisher 的 appstore 规范预检工具对仓库根级文档文件（`README.md` / `README.en.md`）的路径校验缺陷导致的 **CI 工具侧误报**：`README.md` 本身内容与位置均正确。因此本仓库内无需（也无法）通过修改源代码修复，未对 `README.md` 做任何改动。

## 修改的文件
- 无。未修改 `README.md`，也未新增任何文件（严格遵守只允许修改 `pr.changed_files` 的约束）。

## 修复逻辑
### 复现与证据
1. 从上游 EulerPublisher 仓库（`openeuler-mirror/eulerpublisher@master`）拉取了 CI 失败栈中调用的两个文件：
   - `update/container/app/update.py`（失败位置 `update.py:273` → `ContainerVerification.check_code` → `format.check_report`）
   - `update/container/app/format.py`（路径校验实现）
2. 定位到具体缺陷代码：`format._check_all_file_paths()` 调用 `format.parse_image_prefix()`。
   - 对根目录文件 `README.md`，`parse_image_prefix()` 因 `len(contents) == 1` 直接返回 `("", "")`，即 `prefix = ""`。
   - 随后 `DOC_FILES_PATH_FORMAT["README"] = "{0}/README.md"` 经 `.format("", "README.md")` 拼接得到绝对路径 `"/README.md"`。
   - `os.path.exists("/README.md")` 为 False，于是返回 `[Path Error] The expected path should be /README.md`，与 CI 日志逐字一致。
3. 本地用 Python 复现该逻辑，确认 `README.md` 与 `README.en.md` 均会被解析为期望路径 `/README.md`（不存在），必然失败。
4. 对比仓库内镜像目录下的 `README.md`（如 `Cloud/containerd/README.md`）：`parse_image_prefix()` 返回非空 `prefix`，拼出的路径为文件自身路径，因此能通过校验。这证明缺陷仅影响**不在任何镜像最小目录内、位于仓库顶层**的文档文件。

### 根因归属
- 该预检逻辑未对“仓库根目录/顶层非镜像文件”做豁免或跳过（本应在 `parse_image_prefix` 返回空前缀时跳过，而不是拼出 `/README.md`）。这是 EulerPublisher 工具的实现缺陷，属于 CI 工具/基础设施侧问题，而非本 PR 文档内容的问题（与失败分析报告“方向 2”一致）。
- 修复应在上游 EulerPublisher 的 `format.py` 中完成（例如：`parse_image_prefix` 对 `len(contents) == 1` 的文件返回 `("", "")` 后，`check_report` / `_check_all_file_paths` 应跳过空前缀的文件；或显式豁免仓库根级 README 等仓级文档）。

### 为何不在本仓库修改
- 唯一允许修改的文件是 `pr.changed_files` 中的 `README.md`，而失败与 `README.md` 的内容无关，任何内容调整都无法改变工具基于**文件名**的判定（`README` 命中 `DOC_FILES_PATH_FORMAT`，根级路径必然拼成 `/README.md`）。
- 按失败报告“方向 1”将内容迁移到被允许的路径需要**新建文件**，违反“禁止创建任何新文件 / 禁止修改列表外文件”的硬约束。
- 按失败报告“方向 2”的指引，确认属于工具缺陷后应作为 CI 工具侧问题处理，而非改 PR 文档内容。
- 修改 CI 配置来绕过检查属明令禁止操作。

## 潜在风险
无代码改动，无回归风险。说明：在该 EulerPublisher 缺陷被上游修复前，任何仅修改仓库根级 `README.md` / `README.en.md` 的 PR 都会在该 x86-64 预检阶段被误判失败；建议向上游 `openeuler/eulerpublisher` 提交修复，并推动本 PR 以工具侧修复或临时白名单方式重新触发门禁。
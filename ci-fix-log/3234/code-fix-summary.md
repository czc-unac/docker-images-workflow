# 修复摘要

## 修复的问题
无需代码修改：本 PR 为纯文档变更（仅 `README.md`），CI 失败由外部 `eulerpublisher` appstore 发布规范预检对仓库根目录 README 的路径校验缺陷触发，无法通过修改本仓库 `README.md` 内容解决。

## 修改的文件
- （无）

## 修复逻辑

### 根因确认
已拉取上游 `eulerpublisher`（openeuler-mirror 镜像，`master`）源码 `update/container/app/update.py` 与 `update/container/app/format.py` 复核，并在本地复现了校验逻辑：

1. `update.py` 的 `ContainerVerification.check_code()` 调用 `format.check_report(self.change_files)`，本 PR 变更文件为 `["README.md"]`。
2. `format.check_report()` 对每个变更文件执行 `file_type = change_file.split("/")[-1].split(".")[0]`，得到 `"README"`，命中 `DOC_FILES_PATH_FORMAT` 的 `"README"` 键，因此不会被跳过。
3. `format.parse_image_prefix("README.md")` 因路径不含 `/`（`len(contents) == 1`）直接返回 `prefix = ""`。
4. `_check_all_file_paths()` 计算 `correct_path = DOC_FILES_PATH_FORMAT["README"].format("", "README.md")`，结果为绝对路径 `"/README.md"`。
5. `os.path.exists("/README.md")` 为 `False` → 返回 `[Path Error] The expected path should be /README.md`，`fail_count` 累加，与 CI 日志完全一致（本地已验证该分支输出）。

因此：**只要 PR 变更列表中包含仓库根目录的 `README.md`，该预检必然报错**，与 README 的具体正文内容无关。缺陷位于上游校验工具（根目录文件 `prefix` 为空字符串时未跳过，或未将 `prefix="."` 归一化），不在本仓库源码内。

### 为何不做代码修改
- 本任务仅允许修改原始 PR 涉及的 `README.md`。对外部安装的 `eulerpublisher`（`site-packages/eulerpublisher/...`）做任何改动都不在当前源码库中，且 CI 运行环境每次会重新安装该工具，源码库改动无法生效。
- 预检按“变更文件名”判定，`README.md` 的正文内容无法改变其被识别为 `README` 类型、也无法改变 `prefix` 的计算结果；不存在能在 `README.md` 内完成的有效修复。
- 删除/回退本次文档新增会丢弃 PR 的贡献且并非最小化修复，属于掩盖问题，故不采用。
- 分析报告“修复方向”指向的是：该预检应对“非镜像目录的文档类变更”予以豁免——这属于 CI 工具侧修复。

### 建议的上游修复（供人工/工具维护者参考，本次不实施）
在 `format.check_report()` 的 Step 1 中，当 `parse_image_prefix()` 返回的前缀为空（即仓库根目录文件）时跳过路径校验；或对根目录文件使用 `prefix="."`，使 `correct_path` 变为 `./README.md` 并通过 `os.path.exists` 校验。

## 潜在风险
无。本次未对源码仓库做任何改动，不会引入构建或发布风险。原始 PR 的 CI 失败将保持，需由 `eulerpublisher` 侧修复或维护者人工处理。
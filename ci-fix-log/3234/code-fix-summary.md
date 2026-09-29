# 修复摘要

## 修复的问题
无需代码修改：本 PR 的 CI 失败属于 tool/infra-error——`eulerpublisher` 的 appstore 发布规范预检 `format.py` 对**仓库根目录** `README.md` 存在路径归一化缺陷，会把任何对根目录 `README.md` 的改动误判为图片级 README 并要求路径 `/README.md`，该期望路径在文件系统中永不存在，因此任何 README 内容改动都无法通过校验。

## 修改的文件
（无）

## 修复逻辑
1. 分析报告给出的两种方向中，经核对上游 `eulerpublisher` 源码，确认属**方向 2（CI 工具层误报）**，故按流程规则（FIX-05 / `code-fix.py` 的 `no_changes` 分支）不做代码修改。
2. 根因定位（已从上游获取实际源文件验证）：
   - 拉取 `https://gitee.com/openeuler/eulerpublisher/raw/master/update/container/app/format.py`（与 GitHub 镜像 `openeuler-mirror/eulerpublisher/master` 一致）。
   - `check_report()` 第 1 步对每个变更文件：`file_type = change_file.split("/")[-1].split(".")[0]`。对根目录 `README.md`/`README.en.md`，`file_type == "README"`，命中 `DOC_FILES_PATH_FORMAT`，因此**不会**被跳过。
   - `parse_image_prefix("README.md")`：`contents` 只有一个元素，返回 `("", "")`，即 `prefix == ""`。
   - `_check_all_file_paths("README.md")`：`correct_path = DOC_FILES_PATH_FORMAT["README"].format("", "README.md")` → `"/README.md"`；`os.path.exists("/README.md")` 恒为 `False`，于是返回 `[Path Error] The expected path should be /README.md`，`fail_count += 1`，`update.py:273` 判定失败。
   - 在内存中复现该逻辑：`README.md` → `FAIL: expected /README.md (exists=False)`；`README.en.md` → 同样失败。即根目录任何 `README*` 改动都会被误报。
3. 该缺陷与 PR 内容无关：
   - 根目录 `README.md` 是仓库合法的项目主文档，历史上有根 README 文档 PR 合入（如 commit `5c99b50d6`，2026-07-12）。
   - 本 PR 为纯文档变更（根 `README.md` 新增 12 行），被拦截对象正是这个根目录文件。
   - 期望路径 `/README.md` 指向文件系统根目录，逻辑上不可能满足；且该预检在 Docker build / 功能测试之前执行，与镜像构建无关。
4. 在「只允许修改 `README.md`、禁止新增文件、禁止修改 eulerpublisher」的约束下，不存在可让该预检通过的仓库内改法（改动 README 必然重新进入同一误判；把文档移动/合并到其他位置也会触发或需要触碰非授权文件）。强行改动只会掩盖问题，故不做修改。

## 潜在风险
无（未改动任何源码文件）。

## 建议（交由 CI / eulerpublisher 维护方）
在 `update/container/app/format.py` 的 `check_report()` 中，当 `parse_image_prefix(change_file)` 返回空前缀（即文件位于仓库根目录，`len(path.split("/")) == 1`）时跳过该文件的路径校验；或在 `parse_image_prefix` 中对单段路径直接返回可跳过标记。这样根目录 `README.md`/`README.en.md` 的文档 PR 才能正常通过 appstore 发布规范预检。
# 修复摘要

## 修复的问题
经核对上游校验脚本，`README.md` 的 `[Path Error] The expected path should be /README.md` 是 `eulerpublisher` appstore 发布规范预检对**仓库根目录文档文件的误报**（工具侧缺陷），并非本仓库源码问题，无需修改文档内容。

## 修改的文件
- 无（未修改任何文件）

## 修复逻辑

### 核对过程（已实际取源并复现）
1. CI 的 `multiarch/openeuler/x86-64/openeuler-docker-images` 任务通过 `update/container/app/build.sh` 克隆并运行上游 `eulerpublisher`：
   - `git clone https://gitee.com/openeuler/eulerpublisher.git` 后执行 `python3 update/container/app/update.py ... -op check`。
2. 已从 gitee 上游获取该仓库 master（`update/container/app/update.py`、`update/container/app/format.py`），其行号与 CI 日志完全吻合（`update.py:356` 打印 `Difference`，`update.py:273` 输出 specification error）。
3. 用获取到的真实 `format.py` 在本地执行 `check_report(["README.md"])`，**精确复现**了 CI 报错：
   ```
   | README.md | [Path Error] The expected path should be /README.md | FAILURE |
   ```
   而对合法镜像文档路径 `Base/openeuler/README.md` 执行 `check_report`，`fail_count = 0`，校验通过。

### 根因（工具侧缺陷，已定位到代码）
`format.py` 的路径校验逻辑存在缺陷：
- `parse_image_prefix("README.md")` 中 `contents = ["README.md"]`，长度为 1，命中 `if len(contents) == 1: return "", ""`，返回空 prefix（`format.py:123-125`）。
- `check_report` 第 181-193 行对每个变更文件无条件执行路径校验，未跳过空 prefix（即不属于任何场景目录的根级文件）。
- `_check_all_file_paths` 第 248-252 行用空 prefix 拼出 `DOC_FILES_PATH_FORMAT["README"].format("", "README.md")` = `/README.md`；`os.path.exists("/README.md")` 检查的是**文件系统绝对根路径**而非仓库根目录，必然为 False，因而误报路径错误。

即：任何对仓库根目录 `README.md` 的改动都会被该预检判为失败，与文档内容无关。根目录 `README.md` 是仓库自身的合法文件（镜像规范中的 `{prefix}/README.md` 针对的是各镜像目录下的 README）。

### 为什么不在本仓库修复
- 失败由 CI 侧 `eulerpublisher/update/container/app/format.py` 的路径比对缺陷引起（空 prefix / 前导 `/` 未归一化），修复应在上游 `eulerpublisher` 仓库进行。
- 本任务仅允许修改 `pr.changed_files = ["README.md"]`，且禁止新增任何文件。无论怎样编辑根 `README.md`（甚至还原全部改动），只要该文件仍出现在 PR 变更集中，预检都会以同一逻辑报错；改文档内容无法绕过该校验。
- 因此按分析报告“方向 2”结论：根 `README.md` 修改本身合法，失败源于校验脚本路径比对缺陷，判定为工具侧问题并转 infra 处理，不改动文档内容。

## 潜在风险
无。本次未改动任何源码文件，不引入功能或构建风险。建议在上游 `eulerpublisher` 的 `update/container/app/format.py` 中修复：当 `parse_image_prefix` 返回空 prefix 时跳过该文件的路径校验（或对拼接路径做去前导 `/` 的仓库相对路径归一化），以允许仓库根级 README 的合法文档变更。
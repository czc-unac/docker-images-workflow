# 修复摘要

## 修复的问题
经核实，本次 CI 失败属于 EulerPublisher 侧 appstore 规范校验工具对「仅修改仓库根 `README.md` 的纯文档 PR」的**误判（工具缺陷 / infra 类问题）**，与本次文档改动内容无关；**本仓库无需、也无法通过修改 `README.md` 修复**，故未做代码修改。

## 修改的文件
- 无（未修改任何文件，包括 `README.md`）

## 修复逻辑
### 1. 已定位到失败的确切代码路径
通过 gitee 上游仓库获取到 CI 实际执行的校验脚本：

- `eulerpublisher/update/container/app/update.py`（`check_code()` -> `format.check_report(self.change_files)`）
- `eulerpublisher/update/container/app/format.py`（路径校验实现）

### 2. 复现并确认根因（工具缺陷）
本次 PR 的变更文件清单为 `["README.md"]`。`format.check_report()` 的处理链路如下：

1. `file_type = change_file.split("/")[-1].split(".")[0]` -> `"README"`，命中 `DOC_FILES_PATH_FORMAT`（`"README": "{0}/README.md"`），因此进入路径校验；
2. `parse_image_prefix("README.md")`：由于路径只有一段（`len(contents) == 1`），直接返回 `("", "")`，即 `prefix = ""`；
3. `_check_all_file_paths()` 中：
   ```python
   correct_path = DOC_FILES_PATH_FORMAT[type].format(prefix, contents[-1])
   # => "{0}/README.md".format("", "README.md") => "/README.md"
   if not os.path.exists(correct_path):
       return False, f"[Path Error] The expected path should be {correct_path}"
   ```
   即把「仓库根目录的 `README.md`」拼成了**文件系统绝对路径 `/README.md`**，随后 `os.path.exists("/README.md")` 必然为 `False`，从而抛出 `[Path Error] The expected path should be /README.md`，并在 `update.py` 中聚合为 `specification errors for releasing on appstore` 导致构建失败。

本地验证结果：
```
root /README.md exists: False
cwd README.md exists:  True
correct_path: '/README.md'
```
即该校验对**任何修改仓库根 `README.md` 的 PR**都会无条件失败，与被修改的文档内容完全无关（校验只使用 Gitee 返回的文件名，不读取文件内容）。

### 3. 为何不能在本 PR 内修复
- 该校验逻辑位于上游工具 `eulerpublisher`，不在本仓库，且不在本 PR 的 `changed_files=["README.md"]` 允许修改范围内；
- 失败由**文件名**触发，修改 `README.md` 的内容无法让 `os.path.exists("/README.md")` 成立；
- 将说明迁移到 `README.en.md` 同样会命中 `file_type == "README"` 的同一缺陷，且新增/变更其他文件超出本次允许范围；
- 因此不存在任何「只改 `README.md`」的修复方案。按分析报告要求，不应在根因未指向文档内容的情况下对文档做无依据的删改。

### 4. 结论与建议（供人工/上游处理）
- 这是 `eulerpublisher/update/container/app/format.py` 的缺陷：`parse_image_prefix()` 对仓库根级文件返回空前缀后，`_check_all_file_paths()` 未跳过空前缀，误拼出绝对路径 `/README.md`。
- 建议的上游修复方向（不在本仓库、本次不实施）：在校验前跳过 `prefix == ""` 的仓库根级文件，或对根级 `README.md`/`README.en.md` 做白名单豁免。
- 本 PR 的文档改动本身符合项目规范，无需修改。

## 潜在风险
无（本次未对仓库任何文件做改动，不存在引入新问题的风险）。

## 备注
- 分析报告将置信度定为「中」并要求先确认校验工具对 `README.md` 路径的期望语义。本报告已通过阅读上游实际校验源码完成确认：期望路径 `/README.md` 系空前缀拼接产生的错误绝对路径，并非项目规范要求的真实落盘位置。
- 已从上游 gitee 仓库成功获取 `update/container/app/update.py` 与 `update/container/app/format.py` 的实际源码（AtomGit 迁移，gitee raw 仍可访问），并据此完成逻辑复现与本地验证。注意：CI 日志中的行号（`update.py:356`、`:273`）与本地上游 master 版本行号略有偏移，属版本差异，不影响上述根因判定。
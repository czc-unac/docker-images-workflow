# 修复摘要

## 修复的问题
本次 CI 失败为 eulerpublisher appstore 发布规范预检工具对「根目录 README.md 变更」的既有误判（infra-error / CI 工具缺陷），并非本 PR 文档内容本身有误。经上游源码验证后确认：**无需修改本 PR 的任何源码，本 PR 的 README.md 变更不应被回退或改写**。

## 修改的文件
- 无（未修改任何文件；`README.md` 保持原 PR 内容不变）

## 修复逻辑

### 已从上游获取实际源文件验证
- 上游仓库：`https://gitee.com/openeuler/eulerpublisher`（`master`）
- 实际读取文件：
  - `update/container/app/update.py`（blob sha `77fe295d…`，check 操作调用 `format.check_report(self.change_files)`，失败时在 line 273 打印 `There are some specification errors for releasing on appstore in this PR`）
  - `update/container/app/format.py`（blob sha `652e22b4…`）

### 根因（已在源码中复现）
1. `format.check_report()` 遍历变更文件：
   ```python
   file_type = change_file.split("/")[-1].split(".")[0]   # "README.md" -> "README"
   if not os.path.exists(change_file) or file_type not in DOC_FILES_PATH_FORMAT:
       continue
   _, prefix = parse_image_prefix(change_file)
   success, description = _check_all_file_paths(change_file)
   ```
   `DOC_FILES_PATH_FORMAT` 含有键 `"README"`，因此根目录 `README.md` 不会被 `continue` 跳过。
2. `format.parse_image_prefix("README.md")` 中 `contents = ["README.md"]`，`len(contents) == 1`，直接 `return "", ""`，即 `prefix = ""`。
3. `format._check_all_file_paths("README.md")` 计算：
   ```python
   correct_path = DOC_FILES_PATH_FORMAT["README"].format("", "README.md")  # -> "/README.md"
   if not os.path.exists(correct_path):   # "/README.md" 不存在
       return False, "[Path Error] The expected path should be /README.md"
   ```
   于是输出日志中的 `[Path Error] The expected path should be /README.md`，`fail_count` 递增，`check_report` 返回非零，`update.py` 在 appstore 预检阶段报错并以 `FAILURE` 结束。

### 结论
`README.md` 的文件路径与内容都不会影响判定结果——只要根目录 `README.md` 出现在变更集中，该预检就必然把 `prefix` 解析为空串并生成不存在的期望路径 `/README.md`，从而 100% 误报。这与此前分析报告中的"语义疑点"完全吻合，属于 CI 预检工具缺陷（对非镜像类文档、根目录文档缺少豁免/白名单逻辑），修复点位于 eulerpublisher `update/container/app/format.py`，不在本仓库、也不在允许修改的 `pr.changed_files`（README.md）内。

按照本流程约束（只允许修改原始 PR 涉及文件、禁止新建文件），并且分析报告已明确"若确认为 CI 预检工具缺陷……应标记为 infra-error 并由 CI 维护方处理，而非修改本 PR 的文档内容"，故不进行任何代码改动。

### 建议的 CI 侧修复（供 CI 维护方参考，本次不实施）
在 `format.py` 中过滤根目录文档，例如：仅当 `prefix` 非空时才纳入 `DOC_FILES_PATH_FORMAT` 校验，或显式跳过不在场景目录下的顶层 `README.md`（如 `change_file == "README.md"`）。

## 潜在风险
无。本次未修改任何代码，不影响仓库其他功能；风险仅在于该 infra 缺陷未修复前，任何仅修改根目录 `README.md` 的 PR 仍会被同一预检误判失败。
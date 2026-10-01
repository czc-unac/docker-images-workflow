# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（CI 发布规范/路径合规校验失败）
- 置信度: 中
- 知识库匹配: 模式11（元数据/路径校验失败，appstore 路径校验子类）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
2026-09-15 09:16:26,553-...update.py[line:356]-INFO: Difference: [
    "README.md"
]
2026-09-15 09:16:31,262-...update.py[line:273]-ERROR: There are some specification errors for releasing on appstore in this PR, please check as above.
+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+
...
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: CI 校验工具 `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范路径校验逻辑）
- 失败原因: 本次 PR 的 diff 仅包含根目录 `README.md`，该文件被纳入 "appstore 发布规范" 路径校验时被判为 `[Path Error]`——工具的期望路径为 `/README.md`，而实际变更路径记录为 `README.md`（缺少前导斜杠），两者字面不匹配，校验判定为 FAILURE。

### 与 PR 变更的关联
- 有关联。`pr.diff` 只修改了仓库根目录 `README.md`（纯文档，新增 "新增应用镜像可通过 Issue 自动化" 指南，12 行新增、0 删除）。
- 日志 `Difference: [ "README.md" ]` 与 diff 一致，说明正是因为 PR 改动了 `README.md`，该文件才进入 appstore 发布规范的路径校验集合并触发路径错误。
- 该 PR 不含任何 Dockerfile / `meta.yml` / `image-info.yml` / `image-list.yml`，不涉及镜像构建或发布，因此校验对象本身与变更内容语义不匹配。

## 修复方向

### 方向 1（置信度: 中）
CI 的 "releasing on appstore" 规范校验面向镜像发布类文件（新增/变更镜像的路径须符合 `{场景}/{镜像}/{版本}/{OS}/Dockerfile` 层级约定），根目录 `README.md` 这类仓库级文档本不应被纳入该校验。需确认该校验是否应忽略根目录级文档文件，或该 PR 是否本就不应触发 appstore 预检。属门禁工具侧 / 流程问题，Code Fixer 在本仓库内通常无对应可改文件。

### 方向 2（置信度: 低）
若门禁确实要求变更文件路径规范化为前导斜杠形式（期望值 `/README.md`），则当前失败源于路径比较未对 `README.md` 与 `/README.md` 做等价归一化，属 `eulerpublisher` 工具的路径匹配缺陷（infra 性质），与 PR 文档内容无关，Code Fixer 无需修改本仓库。

## 需要进一步确认的点
- `eulerpublisher/update/container/app/update.py` 中第 273 行附近路径校验的期望路径判定逻辑：期望值 `/README.md` 的来源（是否对路径强制加前导 `/`）。
- 该校验是否只应对镜像目录（`Bigdata/`、`AI/`、`Database/` 等场景目录）下的文件生效，根目录 `README.md` 是否应被排除/豁免。
- 是否存在变更文件的忽略清单（如根目录文档、CI 配置等）；该 PR 纯文档变更是否本应跳过 appstore 发布规范检查。
- 日志中未出现 `yaml.parser`、构建或测试错误，确认本次失败仅为该路径校验项，无其他下游 job 失败信息遗漏。

## 修复验证要求
本失败核心疑点在外部门禁工具 `eulerpublisher`（`gitee.com/openeuler/eulerpublisher`）的路径校验逻辑，而非本仓库源文件。若修复方向涉及调整该校验的路径匹配/豁免规则，code-fixer 必须先从 eulerpublisher 对应版本获取 `update/container/app/update.py` 的实际实现，确认期望路径 `/README.md` 的生成方式，并验证修复后对实际传入路径 `README.md` 能正确匹配，再行提交；不得直接假设前导斜杠为唯一根因。

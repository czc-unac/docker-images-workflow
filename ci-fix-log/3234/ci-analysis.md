# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（appstore 发布规范 / 路径预检失败）
- 置信度: 中
- 知识库匹配: 模式11（元数据文件错误 / README 路径校验失败）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
2026-09-15 09:16:26,553-.../eulerpublisher/update/container/app/update.py[line:356]-INFO: Difference: [
    "README.md"
]
...
2026-09-15 09:16:31,262-.../eulerpublisher/update/container/app/update.py[line:273]-ERROR: There are some specification errors for releasing on appstore in this PR, please check as above.
+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范预检逻辑；变更文件由同文件 `line:356` 的 Difference 计算得出）
- 失败原因: PR 的唯一变更文件 `README.md` 未通过 appstore 发布规范的路径校验，校验项报
  `[Path Error] The expected path should be /README.md`，触发规范错误并使 job 失败。

### 与 PR 变更的关联
- 关联成立。日志中 `Difference: ["README.md"]` 表明 CI 预检只针对本 PR 变更的文件进行校验，而本次 PR 恰好只修改了根目录的 `README.md`（新增“新增应用镜像自动化指引”段落）。
- 该校验项对 `README.md` 给出“期望路径应为 `/README.md`”的判定。即：只要 PR 变更集合中出现 `README.md`，预检就会产生该路径错误条目。因此本次失败由 PR 对 `README.md` 的改动直接触发，而非构建/编译/测试代码问题。
- 日志结尾为 `Build step ... marked build as failure` / `Finished: FAILURE`，不存在“成功标志但状态为失败”的情形，故不属于“日志缺失”场景。

## 修复方向

### 方向 1（置信度: 中）
调整 PR 中 `README.md` 的提交路径/位置，使其满足 appstore 发布规范预检所要求的路径（`/README.md` 相对镜像根路径）。即确认该 README 文档应归属的目录层级，将文档内容放入校验器认可的位置，而非直接改动仓库根目录 `README.md`。

### 方向 2（置信度: 低）
若确认根目录 `README.md` 本就应被允许修改，则该失败可能源于 CI 预检工具对 `README.md` 的路径白名单/规范化规则（`update.py` 中的路径比较逻辑）存在缺陷，属于工具侧规则问题，应改由维护者调整预检工具或规范。

## 需要进一步确认的点
1. `eulerpublisher/update/container/app/update.py` 第 273 行周边逻辑，确认 “Check Items: README.md / expected path should be /README.md” 的具体匹配规则（是相对镜像根目录、绝对路径白名单，还是变更文件所在层级校验）。
2. 该预检对“文档类文件（README.md）”的期望存放规则：是否要求 README 必须与 `meta.yml`/`image-info.yml` 等元数据同处镜像最小目录单元，而非仓库根目录。
3. 历史同类案例（模式11，PR #2512 `.claude/agents/README.md`、`.claude/README.md`）所展示的“README 路径不符合规范”是否为同一校验器规则，据此确认修复应“移动文档路径”还是“由工具侧放宽规则”。
4. 由于置信度为“中”，code-fixer 在修改前必须先核对上述 `update.py` 校验逻辑，确认 `README.md` 的合法期望路径后再提交，不得直接假设修复方向 1 成立。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
不适用：本失败为 appstore 路径规范预检，未涉及对第三方/上游源文件的正则 patch。

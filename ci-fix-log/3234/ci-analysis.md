# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（CI 规范/路径静态校验失败）
- 置信度: 中
- 知识库匹配: 模式11
- 新模式标题: （不适用，匹配已有模式）
- 新模式症状关键词: （不适用）

## 根因分析

### 直接错误
```
2026-09-15 09:16:26,553 ... update.py[line:356]-INFO: Difference: [
    "README.md"
]
...
2026-09-15 09:16:31,262 ... update.py[line:273]-ERROR: There are some specification errors for releasing on appstore in this PR, please check as above.
+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: 仓库根目录 `README.md`（CI 校验逻辑位于 `eulerpublisher/update/container/app/update.py:273`，触发差异检测于同文件 `:356`）
- 失败原因: PR 仅改动根目录 `README.md`，CI 的 appstore 发布规范预检将其纳入了变更文件清单（`Difference: ["README.md"]`），随后路径校验判定 `README.md` 不符合期望路径 `/README.md`，导致预检 FAILURE、构建标记失败。

### 与 PR 变更的关联
直接相关。PR 的唯一改动文件就是 `README.md`（新增"通过 Issue 自动化新增镜像"指南），而 CI 报错文件中恰好是 `README.md`。此失败不是构建/编译/测试问题，而是 CI 发布规范路径校验对该文件判定失败。

需要说明的矛盾点：错误信息声称"期望路径应为 `/README.md`"，而该文件本就位于仓库根目录，字面上并不矛盾。这说明校验逻辑对"变更文件应归属的镜像目录路径"存在与文档类改动不兼容的判定，属于 docs-only PR 触发 appstore 规范校验的场景（参考知识库模式11中 README 路径校验类案例）。

## 修复方向

### 方向 1（置信度: 中）
确认该 appstore 规范预检是否应对"纯文档改动（仅 README.md）"的 PR 执行。若属于校验工具对文档类 PR 的误判，应按 CI 侧规则调整（跳过无镜像文件的 PR，或明确 README 类文件的期望路径规则），无需改动本 PR 的文档内容。

### 方向 2（置信度: 低）
若该路径校验是有效规则，则需对照 CI 的期望路径规则 `/README.md`，核对当前变更文件的实际路径表示是否与规则一致（根目录 README 是否被识别为带前导 `/` 的路径），并据此调整文档文件的放置/命名以符合规范。

## 需要进一步确认的点
- `README.md` 的路径校验规则来源与判定逻辑（`eulerpublisher/update/container/app/update.py:273` 及其调用的校验函数），确认"expected path should be /README.md"针对的是文件自身还是其所属镜像目录。
- 该 CI 预检是否只应在变更包含镜像文件（Dockerfile/meta.yml 等）时触发，以及文档类 PR 是否应豁免。
- 错误表中 `Check Items` 与 `Description` 的生成规则，确认 FAILURE 是否由"变更文件不在任一镜像目录内"引起。
- 报告的 `ci.logs` 来自 x86-64 编排/校验 job（`Building remotely on ecs-build-docker-x86-hk`），需确认是否有其它下游架构 job 也复现同一校验失败。

## 修复验证要求
本失败不涉及"修改正则 patch 外部源文件"，无需填写该验证项。

> 备注：本次 `ci.logs` 末尾为 `Finished: FAILURE`，不存在"日志成功但 PR 处于失败状态"的情况，故按上述正常流程分析。若后续确认失败判定来自未提供日志的其它下游 job，则应重新归类为 `infra-error` 并降置信度。

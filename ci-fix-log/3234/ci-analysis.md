# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: lint-error（仓库/发布规范静态校验失败）
- 置信度: 中
- 知识库匹配: 模式11（元数据/规范校验失败，含 README 路径校验历史案例）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
2026-09-15 09:16:26,553 update.py[line:356]-INFO: Difference: [
    "README.md"
]
2026-09-15 09:16:31,262 update.py[line:273]-ERROR: There are some specification errors for releasing on appstore in this PR, please check as above.
+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（appstore 发布规范校验）
- 失败原因: CI 计算出的变更差异为 `README.md`，规范校验对该文件执行路径检查时报 `[Path Error] The expected path should be /README.md`，即校验器期望 README 出现在带前导斜杠的绝对根路径 `/README.md`，而本次改动落在仓库根目录 `README.md`，未通过路径规范。

### 与 PR 变更的关联
本次 PR 为纯文档改动，diff 仅新增 `README.md` 中「新增应用镜像可通过 Issue 由自动化流水线完成」的段落（`new_path: README.md`，新增 12 行）。该文件正是 CI 差异列表中唯一命中并被判定 Path Error 的对象，因此失败由本次 PR 对根目录 `README.md` 的修改直接触发，与镜像构建、代码编译无关。

## 修复方向

### 方向 1（置信度: 中）
确认 CI 对该路径校验的语义：若该校验仅为「上架应用镜像」PR 设计，则纯文档 PR 不应携带根目录 `README.md` 变更进入该检查，可考虑将文档改动与镜像相关 PR 分离，或将该文档内容放入 CI 规范允许的位置（例如对应镜像的最小目录单元内），使路径满足校验器期望。

### 方向 2（置信度: 中）
若经核实仓库根目录 `README.md` 的修改本应被允许（校验器对全局 README 产生误报，`/README.md` 与 `README.md` 仅是路径归一化差异），则属于 eulerpublisher 校验工具的问题（infra-error 性质），Code Fixer 无需修改 Dockerfile，只需由 CI/工具维护方修正路径归一化逻辑。

## 需要进一步确认的点
- 日志不足，需查阅 `eulerpublisher/update/container/app/update.py` 第 273 行附近的 appstore 规范校验逻辑，明确 `[Path Error] The expected path should be /README.md` 的判定规则及其相对根目录。
- 确认该仓库是否允许 PR 仅修改仓库根 `README.md`（对比历史纯文档 PR，如 `AI/diskann/README.md` 的处理结果）。
- 确认 `Difference: ["README.md"]` 的比对基准（是相对仓库根还是相对某个镜像目录），以判断期望路径 `/README.md` 的真实语义。
- 确认是否存在上游 trigger 之外真正失败的架构 job（当前日志本身以 `Finished: FAILURE` 结束，失败发生在 x86-64 job 的规范校验步骤，而非下游未提供 job）。

## 修复验证要求
不涉及正则 patch 外部源文件，无需额外验证步骤。

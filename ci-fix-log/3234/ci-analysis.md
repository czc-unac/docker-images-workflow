# CI 失败分析报告

## 基本信息
- PR: #3234 — docs: add automated new-image request guide
- 失败类型: `lint-error`（appstore 发布规范静态路径校验失败；存在工具误报可能）
- 置信度: 中
- 知识库匹配: 模式11（YAML / 元数据文件错误 —— 含 `.claude/README.md` 等 appstore 路径校验同类历史案例）
- 新模式标题: (命中已有模式，不填)
- 新模式症状关键词: (命中已有模式，不填)

## 根因分析

### 直接错误
```
2026-09-15 09:16:31,262-.../eulerpublisher/update/container/app/update.py[line:273]-ERROR:
There are some specification errors for releasing on appstore in this PR, please check as above.

+-------------+-----------------------------------------------------+--------------+
| Check Items |                     Description                     | Check Result |
+-------------+-----------------------------------------------------+--------------+
|  README.md  | [Path Error] The expected path should be /README.md |   FAILURE    |
+-------------+-----------------------------------------------------+--------------+

2026-09-15 09:16:26,553-.../update.py[line:356]-INFO: Difference: [
    "README.md"
]
```

### 根因定位
- 失败位置: `eulerpublisher/update/container/app/update.py:273`（`_check_specification` 类逻辑；日志行 356 为 diff 计算、行 222 为 fork 仓库 clone 成功）
- 失败原因: CI 侧 appstore 发布规范预检在解析本次 PR 变更文件列表（`Difference: ["README.md"]`）后，判定 `README.md` 的路径不符合发布规范——期望路径为 `/README.md`，而 PR 中该文档以根目录相对路径 `README.md` 提交，`[Path Error]` 校验返回 FAILURE，最终 `Execute shell marked build as failure`。

### 与 PR 变更的关联
- 本 PR 为**纯文档变更**，diff 仅新增 README.md 中 12 行「新增应用镜像 Issue 自动化指南」，未新增/修改任何镜像目录、Dockerfile 或元数据。
- CI 触发方式为 `trigger by note`（PR 评论触发），`update.py` 对该 PR 的 diff 做发布规范校验，唯一变更文件 `README.md` 被纳入校验并报路径错误。因此**失败由本次 README.md 变更直接触发**。
- 但需注意：根目录 README.md 是仓库主文档，本不应作为某个镜像的发布文档参与 appstore 规范校验。报错文案「期望路径应为 `/README.md`」与实际路径 `README.md` 仅差前导斜杠，**高度疑似工具路径归一化缺陷导致的误报**（infra/tooling），也可能该预检要求文档路径必须带前导 `/` 的绝对形式。此点无法仅凭当前日志定论。

## 修复方向

### 方向 1（置信度: 中）
按 appstore 发布规范调整文档存放/命名，使被校验的文档路径与规范期望一致；若根目录 README.md 不在规范允许校验范围内，应将新增指南内容放置到规范约定的文档路径（如既有同类文档所在目录），避免由根 README 触发 appstore 发布预检。

### 方向 2（置信度: 低）
若确认 `README.md` 与 `/README.md` 仅前导斜杠差异为 CI 工具（`update.py` appstore 路径校验）归一化 bug，则属 **infra-error**，与 PR 文档内容无关，Code Fixer 无需改动 PR 正文，应转交 CI/工具维护方修复校验逻辑。

## 需要进一步确认的点
- `eulerpublisher/update/container/app/update.py` 第 273 行附近产生 `[Path Error] The expected path should be /README.md` 的判定规则：期望路径的构造方式与比较逻辑（是否强制前导 `/`、是否只允许特定目录下的文档）。
- appstore 发布规范是否允许根目录 `README.md` 变更；纯文档 PR 是否应跳过该预检。
- `Difference` 是否仅统计本次 diff，是否存在将根文档误判为镜像文档的路径匹配规则。
- 若方向 1 成立，需确认本项目规范中「新增指南类文档」的合法存放路径（例如现有同类文档的实际目录）。

## 修复验证要求
不适用（本失败未涉及正则 patch 第三方/上游源文件）。

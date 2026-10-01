# 修复摘要

## 修复的问题
本次 CI 失败为外部门禁工具 `eulerpublisher` 的 appstore 发布规范路径校验误报（`[Path Error] The expected path should be /README.md`），属基础设施/门禁工具侧问题，本仓库内无需也无法做代码修复。

## 修改的文件
- 无（未修改任何文件）

## 修复逻辑
- 失败类型为 `lint-error`，实际错误来自 CI 校验工具 `eulerpublisher/update/container/app/update.py:273`，该工具对 PR 变更路径 `README.md` 与期望路径 `/README.md` 做字面比较且未做前导斜杠归一化，判定为 FAILURE。
- 本 PR（#3234）仅修改仓库根目录 `README.md`（纯文档，新增 Issue 自动化新增镜像指南 12 行，0 删除），不涉及任何 Dockerfile / `meta.yml` / `image-info.yml` / `image-list.yml`，与 appstore 镜像发布规范无关。
- 分析报告的方向 1、方向 2 均指向门禁工具自身（校验未排除仓库级文档 / 路径未归一化），根因位于外部仓库 `gitee.com/openeuler/eulerpublisher`，而非本仓库 `openEuler/openeuler-docker-images`。
- 允许修改的文件仅 `README.md`。由于该校验针对变更文件路径，在 README.md 内做任何内容修改都无法改变工具对路径 `README.md` 的判定，强行改动只会扩大范围且无法通过门禁，故不做修改。
- 未发现本仓库内存在可调整的校验配置（`config/`、`tests/` 中均无 appstore 路径校验逻辑），确认无对应可改文件。

## 潜在风险
无。未对源码做任何改动，不影响功能。

## 建议（供流程侧处理，非本仓库代码修改）
- 由 `eulerpublisher` 维护方修复路径校验：对变更路径与期望路径统一做前导斜杠归一化，或在 appstore 规范校验中豁免/忽略根目录级文档文件（如 `README.md`、`README.en.md`）。
- 或调整门禁触发条件，使纯文档变更 PR 跳过 appstore 发布规范预检。
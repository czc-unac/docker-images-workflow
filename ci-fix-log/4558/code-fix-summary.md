# 修复摘要

## 修复的问题
本次 CI 失败为基础设施问题（infra-error），无需修改任何代码。

## 修改的文件
- 无

## 修复逻辑
根据 CI 失败分析报告，失败类型为 `infra-error`（模式39：CI 工具依赖缺失），置信度：高。

- 日志显示镜像的 [Build] 与 [Push] 阶段均已完成（`[Build] finished`、`[Push] finished`），失败发生在其后的 eulerpublisher 发布后校验阶段。
- 报错为 `FileNotFoundError: [Errno 2] No such file or directory: 'eulerpublisher'`，位于 `eulerpublisher/update/container/app/update.py:90` 的 `subprocess.call(['eulerpublisher', ...])`。即 CI 运行时环境中缺少 `eulerpublisher` CLI 可执行入口，属于 CI 编排/工具链环境问题，而非本 PR 的镜像构建逻辑。
- 本 PR 仅新增 `HPC/tensorrt/11.3/24.03-lts-sp4/Dockerfile`（`pip3 install tensorrt==11.3`）并同步 `README.md`、`doc/image-info.yml`、`meta.yml`，与失败无因果关系。
- 日志中的 `NoEmptyContinuation: Empty continuation line (line 11)` 仅为 BuildKit 警告，未阻断构建，按最小化原则不予顺带改动。

因此不修改 `pr.changed_files` 中的任何文件，需由 CI 运维侧确保 `eulerpublisher` CLI 入口在构建 agent 的 `PATH` 中可用（或重跑 job 排除临时环境异常）。

## 潜在风险
无
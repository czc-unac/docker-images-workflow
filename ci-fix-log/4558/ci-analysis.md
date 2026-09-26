# CI 失败分析报告

## 基本信息
- PR: #4558 — 【自动升级】tensorrt容器镜像升级至11.3版本.
- 失败类型: infra-error
- 置信度: 高
- 知识库匹配: 模式39（CI工具依赖缺失）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
2026-09-26 07:22:58,223 - INFO - [Build] finished
2026-09-26 07:22:58,223 - INFO - [Push] finished
Traceback (most recent call last):
  File ".../eulerpublisher/update/container/app/update.py", line 367, in <module>
    if obj.check_updates():
  File ".../eulerpublisher/update/container/app/update.py", line 256, in check_updates
    if _check_app_image(file=file) != 0:
  File ".../eulerpublisher/update/container/app/update.py", line 90, in _check_app_image
    if subprocess.call([
  File "/usr/lib64/python3.11/subprocess.py", line 389, in call
    with Popen(*popenargs, **kwargs) as p:
  ...
FileNotFoundError: [Errno 2] No such file or directory: 'eulerpublisher'
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `eulerpublisher/update/container/app/update.py:90`（`_check_app_image` 内部 `subprocess.call(['eulerpublisher', ...])`）
- 失败原因: 镜像的**构建（[Build] finished）与推送（[Push] finished）均已成功完成**，失败发生在 eulerpublisher 工具的**发布后校验阶段**——该阶段以 `subprocess.call([...])` 调用 `eulerpublisher` 可执行文件（CLI 入口）进行 appstore 上架规范校验，但该可执行文件在当前 CI 运行时环境中不存在（`FileNotFoundError: 'eulerpublisher'`）。这属于 CI 编排/工具链自身的环境问题，而非镜像构建逻辑错误。

### 与 PR 变更的关联
- **无关**。PR 仅新增 `HPC/tensorrt/11.3/24.03-lts-sp4/Dockerfile`（`pip3 install tensorrt==11.3`）并同步更新 `README.md`、`doc/image-info.yml`、`meta.yml`。
- 日志中该 Dockerfile 的基础镜像加载、`dnf update`、`dnf install python3-pip`、层导出（`#8 exporting to image ... done`）及镜像推送（`pushing manifest for docker.io/openeulertest/tensorrt:11.3-oe2403sp4-aarch64 ... done`）全部成功。
- 日志中唯一的告警为 `NoEmptyContinuation: Empty continuation line (line 11)`（对应 Dockerfile 中 `RUN` 末尾多出的续行反斜杠产生的空续行），仅为警告，**未阻断构建**，也不是本次失败的根因。
- 因此该失败与 PR 代码改动无因果关系。

## 修复方向

### 方向 1（置信度: 高）
CI 基础设施问题，**Code Fixer 无需修改 Dockerfile 或本 PR 的任何文件**。需由 CI 运维侧修复运行环境：
- 确保 `eulerpublisher` 的 CLI 入口（Python entry point console script）在构建 agent 的 `PATH` 中可用；日志显示 `Successfully installed eulerpublisher-0.0.1.dev321`，但后续以裸命令名 `eulerpublisher` 调用时找不到可执行文件，说明入口脚本未正确安装/未进 `PATH`，或 `_check_app_image` 使用的命令名与安装产物不一致。
- 建议重跑该 job 以排除临时环境异常；如稳定复现，则由 CI 侧修复工具安装路径/PATH。

### 方向 2（可选，置信度: 低）
Dockerfile 第 11 行存在空续行告警，可按项目规范清理该多余的反斜杠以消除 BuildKit `NoEmptyContinuation` 警告，但这是**顺带优化**，即使不处理也不会导致本次 CI 失败，不能作为本失败根因的修复。

## 需要进一步确认的点
1. 确认 `eulerpublisher` 的 CLI 入口在 CI agent 环境中是否可执行（`which eulerpublisher` / 检查 site-packages 的 `entry_points`/`bin` 目录），以判定是 PATH 问题还是包安装不完整。
2. 确认此次 `FileNotFoundError` 是否在其他 PR/构建上稳定复现（若为普遍现象，可确认为 模式39 同类基础设施故障）。
3. 确认下游 x86-64 / aarch64 架构构建 job 是否均在同一后处理步骤失败，还是仅本编排层 job 失败——本日志对应 `aarch64` 构建目录且其 Build/Push 已成功，需比对其它架构 job 结果。

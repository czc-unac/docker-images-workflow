# CI 失败分析报告

## 基本信息
- PR: #4420 — 【软件升级】ceph容器镜像升级至21.3.0版本
- 失败类型: infra-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: CI工具命令缺失
- 新模式症状关键词: FileNotFoundError, eulerpublisher, No such file or directory, subprocess

## 根因分析

### 直接错误
```
2026-09-24 09:20:06,725-...-INFO: The image specification check for releasing on appstore has passed.
Traceback (most recent call last):
  File "/home/jenkins/agent-working-dir/workspace/multiarch/openeuler/aarch64/openeuler-docker-images/eulerpublisher/update/container/app/update.py", line 367, in <module>
    if obj.check_updates():
  File ".../eulerpublisher/update/container/app/update.py", line 256, in check_updates
    if _check_app_image(file=file) != 0:
  File ".../eulerpublisher/update/container/app/update.py", line 77, in _check_app_image
    if subprocess.call([
  File "/usr/lib64/python3.11/subprocess.py", line 389, in call
    with Popen(*popenargs, **kwargs) as p:
  File "/usr/lib64/python3.11/subprocess.py", line 1026, in __init__
    self._execute_child(args, executable, preexec_fn, close_fds,
  File "/usr/lib64/python3.11/subprocess.py", line 1950, in _execute_child
    raise child_exception_type(errno_num, err_msg, err_filename)
FileNotFoundError: [Errno 2] No such file or directory: 'eulerpublisher'
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `eulerpublisher/update/container/app/update.py:77`（`_check_app_image` 函数内的 `subprocess.call([...])`）
- 失败原因: CI 编排工具 `eulerpublisher` 在 `check_updates()` 流程中通过 `subprocess.call(...)` 调用名为 `eulerpublisher` 的可执行文件，但该命令在构建节点的 PATH 中不存在，Python 抛出 `FileNotFoundError`，脚本以非零码退出并被 Jenkins 标记为构建失败。

### 与 PR 变更的关联
无关。日志证据：
1. `Difference` 列表仅列出本 PR 新增/修改的 5 个文件（`Storage/ceph/21.3.0/...`、README、image-info.yml、meta.yml），属于 CI 的正常变更识别。
2. 紧接着日志明确输出 `The image specification check for releasing on appstore has passed.`——即本 PR 的元数据/appstore 规格预检**已通过**。
3. 失败发生在预检通过之后、由 CI 工具自身发起的子进程调用环节，此时尚未进入任何 Docker 构建步骤（日志中无 Dockerfile 构建输出）。

因此该失败与本 PR 的 Dockerfile/entrypoint.sh/元数据内容无因果关系，属于 CI 工具/运行环境问题。

## 修复方向

### 方向 1（置信度: 高）
属于 infra-error，与 PR 代码无关。Code Fixer **不应修改本 PR 的任何文件**。真正需要处理的是 CI 侧：`eulerpublisher` 包被 `pip install ./eulerpublisher` 安装后，未生成（或未暴露）名为 `eulerpublisher` 的命令行入口，或该入口所在目录（通常 `/usr/local/bin`）未包含在 job 的 PATH 中，导致工具自调用失败。应由 CI 维护方在 eulerpublisher 的 pyproject 中补充 console_scripts 入口，或在运行前把安装目录加入 PATH。

### 方向 2（可选）
若确认该入口本应由系统级安装提供，则可能是本 aarch64 构建节点环境残缺（与其他节点不一致）所致，属于基础设施配置差异，同样无需 Code Fixer 处理。

## 需要进一步确认的点
- 确认 `eulerpublisher` 包的 `pyproject.toml` 是否声明了名为 `eulerpublisher` 的 `[project.scripts]` / `console_scripts` 入口点；日志中该包确实 `Successfully installed eulerpublisher-0.0.1.dev321`，因此问题更可能是入口点未注册或不在 PATH，而非包缺失。
- 确认 job 运行环境中 `/usr/local/bin`（pip 安装 console scripts 的默认目录）是否在 PATH 中。
- 对比同期其他 PR 在此 job（`multiarch/openeuler/aarch64/openeuler-docker-images`）上的结果：若同样失败，则确证为环境固有 infra 问题，而非本 PR 引入。
- 注意日志中的触发信息为 `PR 4431 [unac:fix/4420 -> master]`，与上下文 PR #4420 编号不一致，建议确认触发关系（同一修复分支的来源），但不影响上述 infra-error 结论。

## 修复验证要求
不适用（本失败不涉及修改正则 patch 外部源文件，且无需 Code Fixer 修改本 PR）。

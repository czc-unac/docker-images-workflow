# CI 失败分析报告

## 基本信息
- PR: #4295 — 【软件升级】3dslicer容器镜像升级至5.12.4版本
- 失败类型: build-error
- 置信度: 中
- 知识库匹配: 新模式
- 新模式标题: vtkITK目标缺失
- 新模式症状关键词: No rule to make target, vtkITK, gmake, Built target VTK, exit code 2

## 根因分析

### 直接错误
```
#14 10372.3 [100%] Built target ViewsQt
#14 10372.4 [100%] No install step for 'VTK'
#14 10372.4 [100%] Creating '/opt/Slicer-Release/python-install/lib/python3.12/site-packages/vtk-9.6.2.dist-info' directory
#14 10372.4 [100%] Completed 'VTK'
#14 10372.4 [100%] Built target VTK
#14 10372.4 gmake: *** No rule to make target 'vtkITK'.  Stop.
#14 ERROR: process "/bin/sh -c ./build-CTKAppLauncher.sh &&     ./build-tbb.sh &&     ./build-Slicer.sh ${BRANCH} /opt/slicer-arm64.patch" did not complete successfully: exit code: 2
ERROR: failed to solve: process ... did not complete successfully: exit code: 2
```

注：日志中大量 `-- Performing Test ... - Failed` / `-- Check size of ... - failed` 都是 CMake 配置阶段的正常探测（compiler support probe），并非失败根因。日志末尾为 `Finished: FAILURE`，非成功的编排层日志，故不属于“证据不足-基础设施问题”。

### 根因定位
- 失败位置: `HPC/3dslicer/5.12.4/24.03-lts-sp4/build-Slicer.sh:101-103`（第二段 `cmake --build $build_dir --parallel $PARALLEL_JOBS`），由 `HPC/3dslicer/5.12.4/24.03-lts-sp4/Dockerfile:28-30` 的 RUN 触发。
- 失败原因: 第一段 `cmake --build $build_dir --target VTK --parallel 2`（build-Slicer.sh:96-99）成功构建完 VTK 外部工程（`[100%] Completed 'VTK'` / `Built target VTK`）后，紧接着的整项目构建（未指定 `--target`，走默认目标）立即尝试构建名为 `vtkITK` 的目标，而当前生成的构建系统中不存在该目标的规则，gmake 直接以 exit code 2 终止。错误行前的构建耗时约 10372s（约 2.9 小时），且无 `Killed` / `cc1plus ... Killed`、无 timeout 标志，可排除 OOM 与超时。

### 与 PR 变更的关联
本次 PR 为全新镜像版本：新增了 Dockerfile 以及 `build-CTKAppLauncher.sh` / `build-tbb.sh` / `build-Slicer.sh` / `slicer-arm64.patch` 等构建脚本，并更新了 `meta.yml`、`README.md`、`doc/image-info.yml`。其中 `build-Slicer.sh` 采用了“先单独构建 VTK、再全量构建”的两阶段非标准流程。失败正是由该新增的两阶段构建逻辑引入，与仓库中已有镜像（5.8.1）无关，属本次改动直接触发。`meta.yml` 新增条目（`5.12.4-oe2403sp4`）与 README/image-info 的文字改动不是本次失败原因。

## 修复方向

### 方向 1（置信度: 中）
取消“先 `--target VTK` 单独构建、再默认全量构建”的两段式，改为一次性构建 Slicer superbuild 的完整目标（默认全量目标或上游文档指定的完整构建目标），并仅用 `--parallel`（或较低并行度）来规避 OOM。单独对 VTK 调用一次构建很可能使 superbuild 的目标/依赖图处于不一致状态，导致第二次默认构建引用到尚未生成规则的目标。

### 方向 2（置信度: 中）
保留分阶段策略，但第二段构建显式指定正确的目标名（先在配置后执行 `cmake --build $build_dir --target help` 列出实际可用目标，Slicer superbuild 的完整构建目标通常不是裸默认目标，而可能是 `Slicer`），并确认 `vtkITK` 属于内层 Slicer 子构建的模块目标，必须在子项目配置完成后才可被引用。

### 方向 3（置信度: 低）
若确认 `vtkITK` 是内层 `Libs/vtkITK` 模块目标，则第二段应进入内层构建目录（`/opt/Slicer-Release/Slicer-build`）执行构建，而不是在外层 `$build_dir` 顶层执行，从而避免在错误层级引用内层目标。

## 需要进一步确认的点
- 需查阅 Slicer 5.12.4 上游 superbuild（`CMakeLists.txt` / `SuperBuild/`）以确认：默认构建目标的定义、完整构建应用的正确目标名，以及 `vtkITK` 目标由哪一层（VTK 外部工程 vs 内层 Slicer 子构建）生成。
- 需确认“先 `--target VTK` 再默认构建”这一顺序是否确实造成目标图不一致；可在配置完成后用 `cmake --build /opt/Slicer-Release --target help` 验证 `vtkITK` 目标是否存在及其归属层。
- 需获取 aarch64 job 的完整日志，确认该失败在两架构上是否表现一致（当前日志为 x86_64，python 模块路径为 `cpython-312-x86_64-linux-gnu`）。
- 需确认 CTKAppLauncher/TBB 两个前置脚本是否均构建成功（日志显示已进入 Slicer 构建阶段，推测成功，但应核实）。

## 修复验证要求
本修复不涉及“修改正则匹配第三方/上游源文件”，但置信度为“中”，code-fixer 在提交前必须执行以下验证，不得假设修复方向一定正确：
1. 在 Slicer 5.12.4 源码（以 Dockerfile `ARG BRANCH=v5.12.4` 为准，`https://github.com/Slicer/Slicer`）中确认 superbuild 的默认目标与完整构建目标名称，并据此调整 `build-Slicer.sh` 第二段构建命令。
2. 确认 `vtkITK` 目标的生成层级（VTK 外部工程 / 内层 Slicer 子构建 / 单独模块），避免在错误目录层级引用。
3. 如保留“先建 VTK”阶段，需确认该阶段结束后默认/显式全量构建仍能正确解析所有目标；建议以一次干净的完整 Docker 构建验证。
4. 同时核实 aarch64 分支是否同样出现 `No rule to make target 'vtkITK'`，因本修复需覆盖 amd64 与 arm64 两种架构。

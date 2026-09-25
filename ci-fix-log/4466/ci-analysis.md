# CI 失败分析报告

## 基本信息
- PR: #4466 — 【自动升级】3dslicer容器镜像升级至5.12.4版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 脚本缺少执行权限
- 新模式症状关键词: Permission denied, exit code: 126, ./script.sh, chmod +x, mode 100644

## 根因分析

### 直接错误
```
#14 [8/8] RUN ./build-CTKAppLauncher.sh &&     ./build-tbb.sh &&     if [ "amd64" = "arm64" ]; then         BRANCH="main";     fi &&     ./build-Slicer.sh v5.12.4 /opt/zlib.patch
#14 0.081 /bin/sh: line 1: ./build-CTKAppLauncher.sh: Permission denied
#14 ERROR: process "/bin/sh -c ./build-CTKAppLauncher.sh && ..." did not complete successfully: exit code: 126
------
Dockerfile:21
```

### 根因定位
- 失败位置: `HPC/3dslicer/5.12.4/24.03-lts-sp4/Dockerfile:21`
- 失败原因: 构建脚本 `build-CTKAppLauncher.sh`（以及后续的 `build-tbb.sh`、`build-Slicer.sh`）在仓库中不具备可执行权限，Dockerfile 以 `./build-xxx.sh` 方式直接执行时被 shell 拒绝，返回 `Permission denied`（exit code 126）。

### 与 PR 变更的关联
本 PR 为新增镜像版本，diff 显示三个脚本文件均为新建文件，且其 git 模式为 `b_mode: 100644`（普通文件），而非 `100755`（可执行文件）。Dockerfile 第 21 行通过 `./build-CTKAppLauncher.sh && ./build-tbb.sh && ./build-Slicer.sh ...` 直接调用脚本，因此在第一步 `build-CTKAppLauncher.sh` 即失败。该失败由本次 PR 新增内容直接触发，与 PR 变更强相关。

## 修复方向

### 方向 1（置信度: 高）
为 `HPC/3dslicer/5.12.4/24.03-lts-sp4/` 下的 `build-CTKAppLauncher.sh`、`build-tbb.sh`、`build-Slicer.sh` 三个脚本赋予可执行权限（在提交时保留 git 可执行位，或在使用前显式授权）。注意三个脚本都会被 `./` 直接调用，需一并处理，不能只修第一个。

### 方向 2（可选）
若 CI 环境无法保留文件可执行位，可在 Dockerfile 中调用脚本前统一授予执行权限。此为兜底方案，但优先采用方向 1。

## 需要进一步确认的点
- 确认仓库提交时是否保留 Unix 可执行位（部分自动升级工具会以 100644 写入新文件），若如此，仅靠本地 `chmod` 而不修改提交模式无法生效。
- 确认 `zlib.patch` 的行号/内容是否能被目标 `v5.12.4` 版本的 `SuperBuild/External_zlib.cmake` 正确应用（当前日志尚未执行到 `git apply`，属潜在后续风险，非本次失败根因）。

## 修复验证要求
本次修复不涉及正则 patch 外部源文件，无需额外的上游文件正则验证。

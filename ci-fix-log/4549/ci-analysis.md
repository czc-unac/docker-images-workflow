# CI 失败分析报告

## 基本信息
- PR: #4549 — 【自动升级】3dslicer容器镜像升级至5.12.4版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 脚本缺少执行权限
- 新模式症状关键词: Permission denied, exit code: 126, ./build-*.sh, chmod +x

## 根因分析

### 直接错误
```
#14 [8/8] RUN ./build-CTKAppLauncher.sh &&     ./build-tbb.sh &&     if [ "amd64" = "arm64" ]; then         BRANCH="main";     fi &&     ./build-Slicer.sh v5.12.4 /opt/zlib.patch
#14 0.063 /bin/sh: line 1: ./build-CTKAppLauncher.sh: Permission denied
#14 ERROR: process "/bin/sh -c ./build-CTKAppLauncher.sh && ... : exit code: 126
------
Dockerfile:21
  20 |     WORKDIR /opt/
  21 | >>> RUN ./build-CTKAppLauncher.sh && \
```

### 根因定位
- 失败位置: `HPC/3dslicer/5.12.4/24.03-lts-sp4/Dockerfile:21`
- 失败原因: 新增的 `build-CTKAppLauncher.sh` 等脚本被 `COPY` 进镜像后不具备可执行位，`RUN ./build-CTKAppLauncher.sh` 以直接执行方式调用时返回 `Permission denied`（退出码 126），构建在第一个脚本处即终止。

### 与 PR 变更的关联
本次 PR 新增了整个 `HPC/3dslicer/5.12.4/24.03-lts-sp4/` 目录及其 4 个脚本（`build-CTKAppLauncher.sh`、`build-tbb.sh`、`build-Slicer.sh`，以及 `zlib.patch`）。日志显示失败发生在镜像内的 `[8/8] RUN ./build-CTKAppLauncher.sh ...` 步骤，即 PR 新增的构建逻辑本身触发。diff 中这些脚本的模式为 `b_mode: 100644`（无执行权限），而 Dockerfile 未在 `RUN` 前对脚本执行 `chmod +x`，也没有用 `bash script.sh` 的方式调用，导致直接执行时报 `Permission denied`。与 PR 改动直接相关。

## 修复方向

### 方向 1（置信度: 高）
为新增的 shell 脚本补充可执行位并提交（git 中记录为 `100755`），使 `COPY` 进镜像后保留执行权限，`./build-*.sh` 可直接运行。这是在仓库侧最根本的修法。

### 方向 2（置信度: 中）
若不便于修改文件模式，可在 Dockerfile 的 `RUN` 步骤中先对脚本 `chmod +x`（例如对 `/opt/*.sh`），或统一改用 `bash ./build-xxx.sh` 的方式调用，避免依赖可执行位。注意该方式属于 Dockerfile 侧绕过，仍需确认三个脚本链式调用均生效。

## 需要进一步确认的点
- 确认 `build-CTKAppLauncher.sh`、`build-tbb.sh`、`build-Slicer.sh` 三个脚本是否均缺少可执行位（diff 中三者均为 `100644`）；修复需覆盖全部被直接调用的脚本。
- 本 log 在第一个脚本处即失败，后续 `build-tbb.sh`、`build-Slicer.sh` 及 `zlib.patch` 应用能否成功未被执行，需在权限修复后重新触发 CI 验证其是否还存在其他问题（例如 `zlib.patch` 对上游 `External_zlib.cmake` 的 hunk 是否匹配）。
- 若采用方向 2 的 Dockerfile 内 `chmod`，需注意 `zlib.patch` 无需执行权限，仅对 `*.sh` 处理即可。

## 修复验证要求
- 若修复涉及修改脚本文件模式，code-fixer 需确认提交后脚本在 git 索引中为可执行（`git ls-files -s` 显示 `100755`），再触发 CI。
- 若修复涉及在 Dockerfile 中新增 `chmod +x`，请确认该 `chmod` 与脚本 `COPY` 处于同一构建层或之前的层，且路径与 `WORKDIR /opt/` 一致。

# CI 失败分析报告

## 基本信息
- PR: #4631 — 【自动升级】3dslicer容器镜像升级至5.12.4版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 脚本缺少可执行权限
- 新模式症状关键词: Permission denied, exit code 126, ./build-CTKAppLauncher.sh, COPY 未 chmod, no newline at end of file

## 根因分析

### 直接错误
```
#14 [8/8] RUN ./build-CTKAppLauncher.sh &&     ./build-tbb.sh && ...
#14 0.066 /bin/sh: line 1: ./build-CTKAppLauncher.sh: Permission denied
#14 ERROR: process "/bin/sh -c ./build-CTKAppLauncher.sh && ..." did not complete successfully: exit code: 126
------
 > [8/8] RUN ./build-CTKAppLauncher.sh && ...:
0.066 /bin/sh: line 1: ./build-CTKAppLauncher.sh: Permission denied
------
Dockerfile:21
```

### 根因定位
- 失败位置: `HPC/3dslicer/5.12.4/24.03-lts-sp4/Dockerfile:21`（第 8 个构建步骤的 `RUN`）
- 失败原因: Dockerfile 通过 `COPY build-CTKAppLauncher.sh /opt/` 等指令把 shell 脚本拷入镜像后，直接以 `./build-CTKAppLauncher.sh` 方式执行。这些脚本在 PR 中是以普通文件模式（`b_mode: 100644`）新增的，没有可执行位，且 Dockerfile 在 `COPY` 后未执行 `chmod +x`，因此 `/bin/sh` 报 `Permission denied`，进程以 exit code 126 退出，docker build 失败。

### 与 PR 变更的关联
直接由本 PR 引起。PR 新增了 4 个脚本文件（`build-CTKAppLauncher.sh`、`build-tbb.sh`、`build-Slicer.sh` 及 `zlib.patch`），diff 中这些文件的 `b_mode` 均为 `100644`，无执行权限；同时新增的 Dockerfile 第 21 行在 `WORKDIR /opt/` 之后直接用 `./xxx.sh` 调用它们。两者叠加即触发 126 错误。注意 `zlib.patch` 作为数据文件不需要可执行位，只有三个 `.sh` 脚本需要。

## 修复方向

### 方向 1（置信度: 高）
在 Dockerfile 中 `COPY` 脚本之后、执行之前，为三个 `.sh` 脚本补加可执行权限（例如在同一个 RUN 中先 `chmod +x` 再执行，或在 COPY 后单独增加一个 chmod 步骤）。这样无需依赖仓库中文件本身的 mode。

### 方向 2（置信度: 中）
在提交时把 `build-CTKAppLauncher.sh`、`build-tbb.sh`、`build-Slicer.sh` 的文件模式从 `100644` 改为 `100755`（git 可执行位），使 `COPY` 进来即带可执行权限。此方式依赖 Git mode 被正确保留，不如方向 1 稳妥。

### 方向 3（置信度: 低）
不改权限，改为显式用解释器调用，例如 `bash ./build-CTKAppLauncher.sh`。可绕过可执行位，但属于绕行方案。

## 需要进一步确认的点
1. 三个脚本（`build-CTKAppLauncher.sh`、`build-tbb.sh`、`build-Slicer.sh`）应全部补齐可执行权限，不能只修当前报错的第一个。
2. `zlib.patch` 新增内容存在可疑之处：新增的 `set(${proj}_CMAKE_C_FLAGS ${ep_common_c_flags})` 顶格而后续 `if(...)` 又缩进两格，且后续使用 `${${proj}_CMAKE_C_FLAGS}` 嵌套变量展开。权限问题修复后，若构建推进到 `build-Slicer.sh` 的 `git apply zlib.patch` 或 cmake 配置阶段，需确认该 patch 能否被目标 Slicer v5.12.4 源码正确应用（参照模式08：上游行号偏移可能导致 hunk 失败）。这属于权限问题解决后可能暴露的下一层问题，当前日志尚不足以判定。
3. diff 中 `build-CTKAppLauncher.sh` 与 `build-tbb.sh` 末尾为 `\ No newline at end of file`，一般不影响执行，但修复脚本时可一并规整。

## 修复验证要求
本次修复不涉及对第三方/上游源文件的“正则 patch”，无需拉取上游文件验证。但 code-fixer 在提交前应确认：
- 修改后的 Dockerfile 在 `amd64` 与 `arm64` 两个架构路径上，三个 `.sh` 脚本均具备可执行权限（因 arm64 分支会走 `BRANCH=main`，仍执行同一批脚本）。
- 权限修复后需重新运行一次完整构建，确认不再出现 `Permission denied`（exit code 126），并观察是否推进至后续 patch/cmake 阶段的新错误。

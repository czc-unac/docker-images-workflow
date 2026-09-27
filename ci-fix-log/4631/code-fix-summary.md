# 修复摘要

## 修复的问题
修复 Dockerfile 中直接执行 `./build-*.sh` 脚本时因缺少可执行权限导致的 `Permission denied`（exit code 126）构建失败。

## 修改的文件
- `HPC/3dslicer/5.12.4/24.03-lts-sp4/Dockerfile`: 在 `WORKDIR /opt/` 之后、执行脚本之前，新增 `chmod +x build-CTKAppLauncher.sh build-tbb.sh build-Slicer.sh` 并合并进原 `RUN` 步骤。

## 修复逻辑
分析报告根因为：三个 `.sh` 脚本在 PR 中以 `100644`（无可执行位）模式提交，Dockerfile 通过 `COPY` 拷入后直接用 `./xxx.sh` 调用，`/bin/sh` 报 `Permission denied` 并以 126 退出。

采用修复方向 1（置信度最高）：在 Dockerfile 内补加 `chmod +x`，不依赖仓库中文件自身的 mode，因此对 `amd64`（`BRANCH=v5.12.4`）与 `arm64`（`BRANCH=main`）两条路径都生效，三个脚本一次性补齐权限。`zlib.patch` 为数据文件无需可执行位，未处理。

本次修复不涉及对上游源文件的正则 patch，因此无需拉取上游文件做正则验证。

## 潜在风险
- 权限问题解决后，构建会继续推进到 `build-Slicer.sh` 的 `git apply zlib.patch` 及 cmake 配置阶段。该 patch 内容的正确性（缩进、`${${proj}_CMAKE_C_FLAGS}` 嵌套展开、对 Slicer v5.12.4 源码的行号/hunk 匹配）尚待实际构建验证，属于本次权限修复后可能暴露的下一层问题，不在本次最小修复范围内。
- 未改动脚本末尾缺少换行的细节（分析报告指出不影响执行），遵循最小化原则。
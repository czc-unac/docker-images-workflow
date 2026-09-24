# 修复摘要

## 修复的问题
删除 `build-Slicer.sh` 中在 superbuild 顶层不存在的 `--target vtkITK` 构建段，解决 `gmake: *** No rule to make target 'vtkITK'. Stop.` 导致的镜像构建失败（exit code 2）。

## 修改的文件
- `HPC/3dslicer/5.12.4/24.03-lts-sp4/build-Slicer.sh`: 移除第二段 `cmake --build $build_dir --target vtkITK --parallel 2` 及其注释；改为在保留的默认全量构建（使用已按可用内存/CPU 计算出的 `$PARALLEL_JOBS`）中一并构建内层 `Libs/vtkITK`。仅此一处改动，其余逻辑（VTK 预构建、内存上限检测、并行度计算）保持不变。

## 修复逻辑
分析报告的失败点为：单独构建 `VTK` 成功后，紧接着尝试构建 `vtkITK` 目标失败。经从上游 Slicer `v5.12.4` 源码验证（`https://github.com/Slicer/Slicer` tag `v5.12.4`），根因确认为目标层级错误：

1. `CMakeLists.txt:880-883`：`if(Slicer_SUPERBUILD) include(SuperBuild.cmake); return() endif()`。顶层 superbuild 在此提前返回，**不会**执行 `add_subdirectory(Libs)`。
2. `SuperBuild.cmake:541-548`：superbuild 只定义外部工程目标，其中内层工程名为 `Slicer`，`BINARY_DIR=${CMAKE_BINARY_DIR}/${Slicer_BINARY_INNER_SUBDIR}`，即 `/opt/Slicer-Release/Slicer-build`。
3. `Libs/vtkITK/CMakeLists.txt`（`project(vtkITK)`）以及顶层 `CMakeLists.txt:1173 add_subdirectory(Libs)` 仅在内层构建（`Slicer_SUPERBUILD=OFF`）时生效。`vtkITK` 是 **内层 Slicer 子构建** 的目标，不在外层 superbuild 目标图中。

因此 `cmake --build /opt/Slicer-Release --target vtkITK` 必然报 “No rule to make target 'vtkITK'”。这与分析报告“方向 3”一致（`vtkITK` 属内层 `Libs/vtkITK` 模块目标，不应在外层引用）。

由于内层 `Slicer-build` 目录要等外部工程 `Slicer` 被构建时才会配置，无法在外层构建图中单独定位 `vtkITK`。最小且正确的修复是删除该非法目标段，让默认全量构建在内层完成对 `vtkITK` 的编译；OOM 防护由脚本已有的 `detect_memory_limit_mb` + `MEMORY_PER_JOB_MB=8192` 计算出的 `PARALLEL_JOBS` 提供（`cmake --build ... --parallel $PARALLEL_JOBS` 通过 GNU Make jobserver 传递到内层外部工程构建），因而无需再对内层 `vtkITK` 单独降并行。

验证情况：
- `bash -n` 语法检查通过；未引入新的 shellcheck 问题（现存告警均为改动前已有的引号风格问题）。
- 已从上游 Slicer `v5.12.4` 获取 `CMakeLists.txt`、`SuperBuild.cmake`、`Libs/vtkITK/CMakeLists.txt` 确认目标层级，结论明确。
- 分析报告要求的“确认 vtkITK 目标生成层级”已完成：由内层构建生成，非外层目标。

## 潜在风险
- 全量默认构建中，`vtkITK` 的编译并行度取决于 `$PARALLEL_JOBS` 是否能通过外部工程的构建步骤正确传递。GNU Make 通过 `MAKEFLAGS` jobserver 通常可正确传递；若 CI 使用非 Make 生成器或 jobserver 传递失效，内层可能退回更高并行度，理论上仍有 OOM 风险。但这是现有脚本既有的并行度保护机制，本次仅移除非法目标，未改变该机制。
- 保留了 `--target VTK --parallel 2` 预构建段：该段在本次 CI 中已成功执行（日志 `Built target VTK`），不属于失败根因，故未改动。
- 未修改其余文件（Dockerfile、patch、meta.yml、README、image-info.yml），不影响两架构构建流程。
# 修复摘要

## 修复的问题
修复 3dslicer 5.12.4 镜像构建时 Slicer 主构建阶段因并行度过高导致 `cc1plus` 被 OOM 杀死的问题（`Libs/vtkITK/vtkITKGrowCut.cxx` 编译失败）。

## 修改的文件
- `HPC/3dslicer/5.12.4/24.03-lts-sp4/build-Slicer.sh`:
  1. 新增 `detect_memory_limit_mb()`：优先读取容器 cgroup 内存上限（cgroup v2 的 `/sys/fs/cgroup/memory.max`，cgroup v1 的 `/sys/fs/cgroup/memory/memory.limit_in_bytes`），仅在无有限上限时才回退到 `/proc/meminfo` 的 `MemTotal`。
  2. 每编译任务预留内存由 `4096` MiB 提升为 `MEMORY_PER_JOB_MB=8192`（8 GiB），据此计算 `PARALLEL_JOBS`，并保留“不超过 CPU 核数、最小为 1”的约束。
  3. 在预先单独构建 `VTK` 目标之后、主构建之前，新增单独以 `--parallel 2` 构建 `vtkITK` 目标，避免该重内存库与其他目标并发编译。

## 修复逻辑
分析报告指出，失败根因是主构建阶段并行度过高触发 OOM killer（`Killed signal terminated program cc1plus`），并非源码编译错误。原脚本的三处不足均已针对性修复：

- **内存被高估**：原实现用 `/proc/meminfo` 的 `MemTotal` 计算并行度，该值在构建容器内反映的是宿主机总内存，可能远大于容器 cgroup 实际限额。改为优先读取 cgroup 内存上限后，`PARALLEL_JOBS` 依据真实可用内存计算，避免超限。
- **每任务预留内存偏低**：原按 ~4 GiB/任务估算，而 VTK/vtkITK 这类模板实例化密集的翻译单元单进程峰值可超过 4 GiB，故提升到 8 GiB。
- **重内存目标未单独限流**：原脚本仅对 `VTK` 目标做了低并行预构建，而本次 OOM 发生在 `vtkITK` 目标。日志路径 `Libs/vtkITK/CMakeFiles/vtkITK.dir/build.make:236` 表明 `vtkITK` 属于 Slicer 主构建树（非预构建的 VTK 外部工程）。已从上游 `Slicer/Slicer` tag `v5.12.4` 获取 `Libs/vtkITK/CMakeLists.txt` 验证：其通过 `set(lib_name ${PROJECT_NAME})` + `add_library(${lib_name} ${srcs})` 定义了目标 `vtkITK`，故 `--target vtkITK` 目标名正确、可被单独构建。

预期效果：容器无 cgroup 限额时回退宿主机内存；有 32 GiB 限额时主构建为 `--parallel 4`（受 CPU 核数约束时更少），且 `vtkITK` 已提前以 2 路并行构建完成，主构建峰值并行编译的重内存翻译单元数量显著下降。

## 潜在风险
- `detect_memory_limit_mb` 依赖 cgroup 文件路径；若构建环境既无 cgroup v1/v2 限额文件可读，则回退 `/proc/meminfo`（与修复前一致的保守兜底）。已在本地用多种取值（普通数值、`max`、空值、64 位上限值）验证解析正确，脚本 `bash -n` 语法检查通过。
- 单独预构建 `vtkITK` 会使其依赖（ITK、VTK、vtkAddon 等）先被构建，构建总时长可能略有增加；不改变最终产物内容。
- 仅修改了失败直接相关的 `build-Slicer.sh`，未改动 `build-tbb.sh`/`build-CTKAppLauncher.sh`（这两者在本次日志中已构建成功），保持改动最小。
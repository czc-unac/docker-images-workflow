# CI 失败分析报告

## 基本信息
- PR: #4295 — 【软件升级】3dslicer容器镜像升级至5.12.4版本
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 编译OOM被杀
- 新模式症状关键词: Killed signal terminated program cc1plus, c++: fatal error, gmake Error 1, OOM, --parallel, vtkITK

## 根因分析

### 直接错误
（日志来自构建 job `#14 RUN ./build-CTKAppLauncher.sh && ./build-tbb.sh && ./build-Slicer.sh v5.12.4 /opt/slicer-arm64.patch`）

```
#14 12275.1 c++: fatal error: Killed signal terminated program cc1plus
#14 12275.1 compilation terminated.
#14 12275.1 gmake[5]: *** [Libs/vtkITK/CMakeFiles/vtkITK.dir/build.make:236: Libs/vtkITK/CMakeFiles/vtkITK.dir/vtkITKGrowCut.cxx.o] Error 1
#14 12275.1 gmake[5]: *** Waiting for unfinished jobs....
#14 12275.1 gmake[4]: *** [CMakeFiles/Makefile2:6565: Libs/vtkITK/CMakeFiles/vtkITK.dir/all] Error 2
#14 12275.1 gmake[3]: *** [Makefile:156: all] Error 2
#14 12275.1 gmake[2]: *** [CMakeFiles/Slicer.dir/build.make:90: Slicer-prefix/src/Slicer-stamp/Slicer-build] Error 2
#14 12275.1 gmake[1]: *** [CMakeFiles/Makefile2:1629: CMakeFiles/Slicer.dir/all] Error 2
#14 12275.1 gmake: *** [Makefile:91: all] Error 2
#14 ERROR: process "/bin/sh -c ./build-CTKAppLauncher.sh &&     ./build-tbb.sh &&     ./build-Slicer.sh ${BRANCH} /opt/slicer-arm64.patch" did not complete successfully: exit code: 2
```

### 根因定位
- 失败位置: Slicer 构建阶段目标 `Libs/vtkITK`，具体文件 `vtkITKGrowCut.cxx.o`（对应 `Libs/vtkITK/CMakeFiles/vtkITK.dir/build.make:236`），由 `build-Slicer.sh` 中 `cmake --build $build_dir --parallel $PARALLEL_JOBS` 触发
- 失败原因: 编译器进程 `cc1plus` 被内核信号杀死（`Killed signal terminated program cc1plus`），即构建期内存耗尽被 OOM killer 终止，**并非源码语法/编译错误**。日志显示构建已推进到 74%~98%，且 ITK、CTK、HDF5、GDCM、vtkITK 等多个目标在并发编译（`gmake[5]: Waiting for unfinished jobs....`），说明实际并行度仍然很高，内存峰值超出可用上限。

### 与 PR 变更的关联
该 PR 为新增镜像版本，新增了 `Dockerfile`、`build-Slicer.sh`、`build-tbb.sh`、`build-CTKAppLauncher.sh`、`slicer-arm64.patch` 等文件，失败 job 正是执行这些新增脚本的构建步骤，因此失败与本 PR 直接相关。

值得注意的是，PR 作者已经预见到内存问题并在 `build-Slicer.sh` 中加入了缓解逻辑（注释明确写明 "the compiler (cc1plus) is OOM-killed"）：
- `TOTAL_MEMORY_MB=$(awk '/^MemTotal:/ {printf "%d", $2 / 1024}' /proc/meminfo)`
- `PARALLEL_JOBS=$(( TOTAL_MEMORY_MB / 4096 ))`，并与 CPU 核数取小
- 先单独以 `--parallel 2` 构建 `VTK` 目标

但该缓解**不充分**：日志证明 OOM 仍然发生在主构建阶段的 `vtkITK` 目标（`vtkITKGrowCut.cxx` 属于 vtkITK，不在预先单独构建的 `VTK` 目标范围内）。疑似原因：
1. `/proc/meminfo` 的 `MemTotal` 反映的是宿主机总内存，而 Docker 构建容器可能受 cgroup 内存上限约束；容器内存远小于宿主机时，按宿主机内存计算的 `PARALLEL_JOBS` 会严重高估可用内存。
2. 每个编译任务按 ~4 GiB 估算可能偏低，vtkITK / VTK 这类模板实例化密集的翻译单元单进程内存占用可远超 4 GiB。
3. 仅对 `VTK` 目标做了低并行保护，未对同样耗内存的 `vtkITK` 目标做类似处理。

## 修复方向

### 方向 1（置信度: 高）
降低主构建阶段的并行度并对 vtkITK 类重内存目标单独限流，使峰值内存不超限。具体思路（不含代码）：
- 将 `PARALLEL_JOBS` 的计算依据由 `/proc/meminfo`（宿主机总内存）改为**容器 cgroup 内存上限**（如读取 `/sys/fs/cgroup/memory.max` 或 `/sys/fs/cgroup/memory/memory.limit_in_bytes`，并按是否存在回退），避免高估可用内存。
- 大幅提高每编译任务的预留内存（例如由 ~4 GiB 提升到更保守的值），必要时将 Slicer 主构建直接降为 `--parallel 1`/`--parallel 2`。
- 参照现有对 `VTK` 目标的处理，把 `vtkITK`（以及其它已知内存大户）也改为先单独以低并行度构建，再执行整体构建。

### 方向 2（置信度: 中）
从构建环境侧增加可用内存或拆分构建层：
- 提高/取消 Docker 构建容器的内存上限（runner 侧），或在内存更大的 runner 上构建。
- 将 Slicer 大构建拆分为更多独立 `RUN`/缓存层，使网络或单点失败后无需整体重建，并便于对最重的目标单独控制并行度。
- 确认是否可通过 CMake 选项关闭不需要的模块以降低编译内存压力。

## 需要进一步确认的点
1. CI 构建容器的实际内存上限是多少？`/proc/meminfo` 在构建容器内显示的是宿主机内存还是容器限额？这是判定 `PARALLEL_JOBS` 是否被高估的关键（日志本身不包含该信息）。
2. `vtkITKGrowCut.cxx` 单进程编译的实际峰值内存需求，用以校准"每任务预留内存"取值。
3. 是否所有架构构建（amd64/arm64）都在同一阶段失败，还是仅某一架构内存受限更严重（本次提供的日志未标注具体架构 job）。
4. 提供的日志中 `#14` 构建层是否即为真正失败的架构 job；需确认是否还存在独立的 `/job/x86-64/…` 或 `/job/aarch64/…` 下游 job 日志（当前日志末尾为 `Finished: FAILURE`，属真实构建失败，但仍建议交叉核对各架构 job）。

## 修复验证要求
本次失败不属于"正则 patch 外部源文件"场景，无需上游文件正则匹配验证。但鉴于本修复涉及构建资源计算逻辑（置信度虽高，具体阈值仍需实测确认），code-fixer 在提交前应：
1. 从上游 Slicer v5.12.4（`BRANCH=v5.12.4`）确认 `vtkITK` 目标/`vtkITKGrowCut.cxx` 确实属于主构建而非预先单独构建的 `VTK` 目标，确保限流目标选择正确。
2. 在目标基础镜像 `openeuler/openeuler:24.03-lts-sp4` 中确认 cgroup 版本与内存限额文件路径（cgroup v1 的 `memory/memory.limit_in_bytes` 与 cgroup v2 的 `memory.max` 不同），确保新逻辑在 CI 容器内能正确读取内存上限。
3. 说明修复后预期的峰值并行度与内存占用量级，避免仅调参数而无依据。

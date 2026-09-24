# CI 失败分析报告

## 基本信息
- PR: #4420 — 【软件升级】ceph容器镜像升级至21.3.0版本
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 编译进程OOM被杀
- 新模式症状关键词: `Killed signal terminated program cc1plus`, `fatal error`, `ninja: build stopped`, OOM, ceph, rgw_rados.cc, MemTotal

## 根因分析

### 直接错误
```
#12 5766.9 [1055/1775] Building CXX object src/rgw/CMakeFiles/rgw_common.dir/driver/rados/rgw_rados.cc.o
#12 5766.9 FAILED: [code=1] src/rgw/CMakeFiles/rgw_common.dir/driver/rados/rgw_rados.cc.o
#12 5766.9 g++: fatal error: Killed signal terminated program cc1plus
#12 5766.9 compilation terminated.
#12 5777.6 ninja: build stopped: subcommand failed.
#12 ERROR: process "... && ninja -j\"$JOBS\" && ninja install" did not complete successfully: exit code: 1
```
日志中其余 `warning`（`-Wstringop-overflow`、`-Wrestrict`、`-Wmismatched-new-delete`）均为**非致命告警**，且编译命令未使用 `-Werror`（仅 `-Werror=format-security`、`-Werror=vla`），因此**不是根因**。真正致命信号是 `g++: fatal error: Killed signal terminated program cc1plus`，即编译器进程 `cc1plus` 被内核 OOM killer 杀死，导致 `ninja` 报错退出（构建镜像的 Dockerfile 自身注释也印证了该风险：*"Ceph contains very memory-hungry translation units ... so that the compiler (cc1plus) is not OOM-killed"*）。

### 根因定位
- 失败位置: `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile:44-55`（`RUN git clone ... && ninja -j"$JOBS" && ninja install` 步骤），OOM 发生在编译 `src/rgw/driver/rados/rgw_rados.cc` 时
- 失败原因: 内存不足，`cc1plus` 被 OOM killer 杀死。Dockerfile 中的内存限流逻辑存在问题：它通过 `/proc/meminfo` 的 `MemTotal` 计算 `MAX_JOBS=$(( MEM_MB / 4096 ))`，而容器内 `/proc/meminfo` 默认反映的是**宿主机总量**而非容器 cgroup 内存上限，导致 `MAX_JOBS` 被高估、实际并行编译进程数超出容器可用内存，在编译 `rgw_rados.cc`（该 PR 注释自承是内存消耗极大的翻译单元）时触发 OOM。

### 与 PR 变更的关联
本次失败**完全由本 PR 新增的 Dockerfile 引起**。`Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile` 为新增文件（`new_file: True`），构建在 `[6/8]` 步骤失败，失败位置正是该新增 Dockerfile 的 `ninja -j"$JOBS"` 步骤。PR 已经意识到 Ceph 编译的内存压力并加入了基于内存的并行度限制，但该限制实现不足以避免 OOM。其余变更（`entrypoint.sh`、`README.md`、`image-info.yml`、`meta.yml`）均未进入该失败步骤。

## 修复方向

### 方向 1（置信度: 高）
修正内存限流逻辑，使并行度真正匹配容器可用内存：
- 读取容器 cgroup 内存上限（如 cgroup v2 `/sys/fs/cgroup/memory.max`，或 cgroup v1 `/sys/fs/cgroup/memory/memory.limit_in_bytes`）而非宿主机 `/proc/meminfo` 的 `MemTotal`；当读到 `max`/极大值时再回退。
- 同时将每 job 的内存估算值调大（当前按 ~4 GiB/job 估算，而 `rgw_rados.cc`、`rgw_lc.cc` 实测需求更高），确保 `JOBS` 足够保守。
- 可选：为 `ninja` 增加基于负载/内存的并发约束（如 `-l`、或直接使用固定较低的 `-j`），并考虑给构建阶段分配更大内存或加入 swap。

### 方向 2（可选）
若无法可靠读取 cgroup 限制，则直接将该 RUN 步骤的 `JOBS` 固定为保守值（例如 2~4）或按经验值硬编码上限，牺牲构建时长换取构建成功率。

## 需要进一步确认的点
1. 构建容器实际可用的内存上限（cgroup 限制）与宿主机内存，以及 `nproc` 的实际返回值，用以确定安全的 `JOBS`/每-job 内存阈值。
2. 读者如需确认 `rgw_rados.cc` 的峰值内存占用，需在同等环境复现构建（本报告仅基于日志，日志已被截断，未提供 `do_cmake.sh`/configure 阶段输出，但失败的 `[6/8]` 步骤与 `[1055/1775]` 进度足以定位为编译阶段 OOM）。
3. 是否需要针对 aarch64 与 x86-64 分别调整并行度（日志未区分架构，但两架构构建机内存可能不同）。

## 修复验证要求
本修复不涉及 patch 第三方源文件的正则匹配，无需上游文件校验。但 code-fixer 在提交前应确认：
- 新的内存/并行度计算方式在容器内能正确读取到 cgroup 内存上限（若非固定值方案），且计算出的 `JOBS` 至少为 1；
- 保证 `ninja -j"$JOBS"` 在低内存构建机上不会再次触发 `cc1plus` OOM。

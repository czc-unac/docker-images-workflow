# CI 失败分析报告

## 基本信息
- PR: #4263 — 【问题修复】ceph容器镜像升级至21.3.0版本
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 编译OOM内存耗尽
- 新模式症状关键词: Killed signal terminated program cc1plus, ninja: build stopped, -j$(nproc), OOM

## 根因分析

### 直接错误
```
#12 4372.9 FAILED: [code=1] src/osd/CMakeFiles/osd.dir/OSD.cc.o
#12 4372.9 g++: fatal error: Killed signal terminated program cc1plus
#12 4372.9 compilation terminated.
#12 4386.0 FAILED: [code=1] src/osd/CMakeFiles/osd.dir/PG.cc.o
#12 4386.0 g++: fatal error: Killed signal terminated program cc1plus
#12 4395.9 FAILED: [code=1] src/osd/CMakeFiles/osd.dir/PrimaryLogPG.cc.o
#12 4395.9 g++: fatal error: Killed signal terminated program cc1plus
#12 4497.0 ninja: build stopped: subcommand failed.
#12 ERROR: process "... && ninja -j$(nproc) && ninja install" did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile:41-46`（`RUN git clone ... && ninja -j$(nproc) && ninja install`）
- 失败原因: 编译过程中 `g++` 前端 `cc1plus` 被以 `Killed signal`（SIGKILL）终止，这是典型的容器内存不足被内核 OOM Killer 杀死的特征，而非源码语法/链接错误。构建使用 `ninja -j$(nproc)` 以全部 CPU 核数并行编译，而 Ceph 的 OSD 相关大型翻译单元（`OSD.cc`、`PG.cc`、`PrimaryLogPG.cc`，均带 `-O3` 优化）单个编译即占用大量内存，多进程并发叠加后超出容器可用内存，OOM Killer 连续杀死多个 `cc1plus`，ninja 遂停止构建。
- 佐证: 多个不同的 `.cc` 编译目标均以完全相同的 `Killed signal terminated program cc1plus` 失败，而不是某个具体文件的语法错误，说明失败与内存压力相关而非代码本身。

### 与 PR 变更的关联
本 PR 新增 `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`，其中第 45 行使用无上限的 `ninja -j$(nproc)` 并行编译。该改动直接引入了本次 OOM 失败，属于 PR 自身引入的问题。

### 非根因说明
日志中的以下信息均为非致命告警，不是失败根因：
- `FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 2)` — 仅是大小写风格警告。
- `UndefinedVar: Usage of undefined variable '$LD_LIBRARY_PATH' (line 48)` — 仅是 BuildKit 变量自引用告警。
- 大量 `-Wstringop-overflow=` / `-Wrestrict` / `-Wmismatched-new-delete` 警告 — 均为编译器告警，未转为错误（未使用 `-Werror`）。

## 修复方向

### 方向 1（置信度: 高）
限制并行编译任务数，降低同时运行的 `cc1plus` 进程数量以避免内存耗尽。例如将 `ninja -j$(nproc)` 改为较小的固定并发（如 `-j4` 或 `-j$(( $(nproc) / 2 ))`），并可在配置/构建阶段进一步降低编译内存占用（如减少并行度、分步构建或用更少内存的优化等级）。具体实现由 Code Fixer 决定。

### 方向 2（置信度: 中）
若 CI runner 内存上限确实过低，可考虑在构建环境中增加可用内存/swap，或改为分阶段编译（如先构建核心目标再整体安装），但首选仍是限制 `ninja` 并发数这一最小改动。

## 需要进一步确认的点
1. 该 CI runner 容器的内存上限与 CPU 核数（决定 `nproc` 的取值），以确认并发数与内存的匹配关系。
2. 对比同目录下已成功构建的 `Storage/ceph/20.3.0/24.03-lts-sp4/Dockerfile` 的 `ninja` 并发参数，确认历史可用配置作为参照。

## 修复验证要求
本问题的修复方向不涉及对第三方/上游源文件施加正则 patch，无需额外上游文件正则验证。Code Fixer 只需在本地/CI 重新触发该 Dockerfile 构建，确认 `ninja` 阶段不再出现 `Killed signal terminated program cc1plus` 且构建成功即可。

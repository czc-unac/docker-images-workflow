# 修复摘要

## 修复的问题
限制 Ceph 编译阶段的 `ninja` 并行任务数，避免容器内存耗尽触发内核 OOM Killer 杀死 `cc1plus` 导致构建失败。

## 修改的文件
- `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`: 将第 45 行的 `ninja -j$(nproc)` 改为 `ninja -j4`。

## 修复逻辑
CI 分析报告定位根因为编译 OOM：`g++: fatal error: Killed signal terminated program cc1plus`，多个大型翻译单元（`OSD.cc`、`PG.cc`、`PrimaryLogPG.cc`）在 `-O3` 下同时编译，`ninja -j$(nproc)` 会按全部 CPU 核数并行启动 `cc1plus`，单个进程占用内存很大，并发叠加后超出容器可用内存，被 OOM Killer 连续杀死，导致 `ninja: build stopped`。

按照分析报告的“方向 1（置信度: 高）”，采用限制并发数这一最小改动：将无上限的 `-j$(nproc)` 改为固定 `-j4`，显著降低同时运行的编译进程数量，从而压低峰值内存占用。改动仅涉及 ceph 的 `ninja` 调用，未触碰其他无关代码。本问题不涉及对上游源文件的正则 patch，无需上游正则验证。

## 潜在风险
- 固定 `-j4` 会在 CPU 核数较多的 runner 上降低编译并行度，编译耗时可能增加，但可换取构建稳定性。
- 若 runner 内存极低（如小于约 8GB），`-j4` 仍有可能 OOM，此时可进一步下调为 `-j2` 或 `-j1`。
- `libnbd` 阶段仍保留 `make -j$(nproc)`（该阶段在此之前已成功构建，未出现 OOM），为遵循最小化原则未作改动。
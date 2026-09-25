# 修复摘要

## 修复的问题
blat 2.5.1 镜像构建时 `git clone -b 2.5.1` 从错误的仓库（`icebert/pblat-cluster`，该仓库没有 2.5.1 ref）拉取源码导致 `fatal: Remote branch 2.5.1 not found`（exit 128）。

## 修改的文件
- `HPC/blat/2.5.1/24.03-lts-sp4/Dockerfile`: 将 clone 源从 `https://github.com/icebert/pblat-cluster.git` 改为 `https://github.com/icebert/pblat.git`；工作目录由 `/blat-cluster` 改为 `/blat`；`cp pblat-cluster /usr/bin/blat` 改为 `cp pblat /usr/bin/blat`。
- `HPC/blat/doc/image-info.yml`: 将 `homepage` 修正为 `https://github.com/icebert/pblat/`，`upstream.version_url` 由不存在的 `icebert/blat-cluster` 修正为 `icebert/pblat`（消除自动升级脚本目标仓库错误这一根因）。
- `HPC/blat/README.md`: 将 "Learn more at" 链接更新为实际上游 `https://github.com/icebert/pblat/`。

## 修复逻辑
分析报告方向 1 要求核实上游 2.5.1 对应的真实 ref。经 `git ls-remote` 核实：

- `icebert/pblat-cluster`：仅有 tag `0.1/0.2/0.3/1.0/1.1`，**不存在 2.5.1**；
- `icebert/pblat`：存在 tag `2.5`、`2.5.1`。

即 2.5.1 是 `icebert/pblat`（多线程并行版）的版本，而本次自动升级新增的 Dockerfile 直接从 1.1（cluster/MPI 版）的 Dockerfile 复制，仍保留了 `pblat-cluster` 仓库与 `pblat-cluster` 产物名，造成 clone 失败。修复方式是把 clone 指向真正包含 2.5.1 的 `icebert/pblat` 仓库，并把产物 `pblat` 复制为 `/usr/bin/blat`；原有 sed 补丁（`MACHTYPE`、`-fcommon`、`htmshell.c`）经确认在 pblat 2.5.1 源码中均能匹配，未做改动。

同时 `image-info.yml` 中 `upstream.version_url` 被此前的 "Just blat" 改动误写成不存在的 `icebert/blat-cluster`，这正是自动升级脚本选中错误上游版本的根因，已修正为 `icebert/pblat` 以免后续再次生成同类无效升级 PR。

**验证结果**：已通过 `git ls-remote https://github.com/icebert/pblat.git` 确认 `refs/tags/2.5.1` 存在；并从该 tag 全新建 clone，按 Dockerfile 中完全相同的 sed 序列执行 `make -j$(nproc)`，编译成功（exit 0），生成 `pblat` 可执行文件。

## 潜在风险
- 该镜像历史上 1.1 标签为 `icebert/pblat-cluster`（MPI 集群版），而 2.5.1 实际来源为 `icebert/pblat`（多线程版）。`README.md` / `image-info.yml` 中"集群并行、MPI"的描述文字与 `dependency: MPI` 未改动，以保持最小化修改；由于 1.1 仍为集群版、2.5.1 为多线程版，同一镜像内描述只能大致兼容。若维护者希望彻底区分，建议后续单独将多线程版拆分为独立镜像或补充说明。
- Dockerfile 仍安装 `openmpi-devel` 并设置 `ENV PATH=/usr/lib64/openmpi/bin:$PATH`，对 pblat 2.5.1 属于无害冗余，未删除以缩小改动范围。
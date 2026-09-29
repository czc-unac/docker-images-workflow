# 修复摘要

## 修复的问题
blat 2.5.1 镜像构建时 `git clone -b 2.5.1` 从错误的仓库（`icebert/pblat-cluster`，该仓库不存在 `2.5.1` ref）拉取源码，导致 `fatal: Remote branch 2.5.1 not found in upstream origin`（exit code 128）；将构建源改指向真正包含 `2.5.1` 的 `icebert/pblat` 仓库后修复。

## 修改的文件
- `HPC/blat/2.5.1/24.03-lts-sp4/Dockerfile`: clone 源由 `https://github.com/icebert/pblat-cluster.git` 改为 `https://github.com/icebert/pblat.git`；`WORKDIR` 由 `/blat-cluster` 改为 `/blat`；产物由 `cp pblat-cluster /usr/bin/blat` 改为 `cp pblat /usr/bin/blat`。
- `HPC/blat/doc/image-info.yml`: `upstream.version_url` 由不存在的 `icebert/blat-cluster` 修正为 `icebert/pblat`，`homepage` 由 `https://github.com/icebert/pblat-cluster/` 修正为 `https://github.com/icebert/pblat/`。
- `HPC/blat/README.md`: "Learn more at" 链接由 `https://github.com/icebert/pblat-cluster/` 更新为 `https://github.com/icebert/pblat/`。

> 未修改 `HPC/blat/meta.yml`：`2.5.1-oe2403sp4` 条目及其路径本就正确，无需改动（遵循最小化原则）。

## 修复逻辑
分析报告方向 1 要求核实上游 `2.5.1` 对应的真实 ref，方向 2 指出 `image-info.yml` 中 `version_url`（`icebert/blat-cluster`，无 `p`）与 Dockerfile 克隆地址（`icebert/pblat-cluster`，有 `p`）不一致。经上游核实（`git ls-remote` / GitHub API）：

- `icebert/pblat-cluster`：仅有 tag `0.1 / 0.2 / 0.3 / 1.0 / 1.1`，**不存在 `2.5.1`**；
- `icebert/blat-cluster`：仓库不存在（404）；
- `icebert/pblat`：存在 tag `2.5`、`2.5.1`。

即 `2.5.1` 是 `icebert/pblat`（多线程并行版 blat）的版本号；本次自动升级新增的 `2.5.1/` Dockerfile 从 1.1（cluster/MPI 版）的 Dockerfile 复制而来，仍保留了 `pblat-cluster` 仓库与 `pblat-cluster` 产物名，因此 `git clone -b 2.5.1` 必然失败。修复方式是把构建指向真正包含 `2.5.1` 的 `icebert/pblat`，并把产物 `pblat` 复制为 `/usr/bin/blat`；同时把 `version_url`/`homepage` 修正为 `icebert/pblat`，消除自动升级脚本取到错误上游（`blat-cluster` 404 后落到 `pblat`）的根因，避免后续再次生成同类无效升级 PR。原有 sed 补丁（`MACHTYPE`、`-fcommon`、`htslib/Makefile`、`lib/htmshell.c`）与 pblat 2.5.1 源码兼容，未做改动。

**上游与构建验证结果**：
1. 已通过 `git ls-remote --tags https://github.com/icebert/pblat.git` 确认 `refs/tags/2.5.1` 存在；并确认 `icebert/pblat-cluster` 无 `2.5.1`。
2. 已从 `icebert/pblat` 的 `2.5.1` tag 拉取源码，按 Dockerfile 中完全相同的 sed 序列处理后执行 `make -j$(nproc)`，编译成功（exit 0），生成可执行文件 `pblat`。
3. 本修复不涉及在提交阶段用正则 patch 第三方源文件（sed 在 Docker 构建时作用于 clone 下来的源码），无需额外正则匹配验证；上述 sed 目标行（`MACHTYPE=x86_64`、`CFLAGS=-O -Wall`、`lib/htmshell.c` 的 `format != NULL && args != NULL`）均已在 pblat 2.5.1 源码中确认存在。

## 潜在风险
- 该镜像历史 `1.1` 标签来源于 `icebert/pblat-cluster`（MPI 集群版），而 `2.5.1` 实际来源于 `icebert/pblat`（多线程版）。`README.md` / `image-info.yml` 中"集群并行、MPI"的描述文字与 `dependency: MPI` 未改动以保持最小化修改；同一镜像下两种版本描述只能大致兼容。若维护者希望区分，建议后续单独拆分多线程版镜像或补充说明。
- Dockerfile 仍安装 `openmpi-devel` 并设置 `ENV PATH=/usr/lib64/openmpi/bin:$PATH`，对 pblat 2.5.1 属于无害冗余，未删除以缩小改动范围。
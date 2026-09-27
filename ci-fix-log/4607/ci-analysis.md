# CI 失败分析报告

## 基本信息
- PR: #4607 — 【自动升级】cesm容器镜像升级至2.1.5版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: cp目标目录缺失
- 新模式症状关键词: `cp: cannot create regular file`, `No such file or directory`, `micro_mg3_0.F90`, `pumas`, Dockerfile RUN 步骤

## 根因分析

### 直接错误
```
#18 [12/13] RUN git clone https://github.com/ESCOMP/ESCOMP-Containers.git /containers && \
        cp -rf /containers/CESM/2.2/Files/... && \
        cp -rf /containers/CESM/2.2/Files/micro_mg3_0.F90 /opt/ncar/cesm2/components/cam/src/physics/pumas/micro_mg3_0.F90 && \
        ...
#18 0.057 Cloning into '/containers'...
#18 0.951 cp: cannot create regular file '/opt/ncar/cesm2/components/cam/src/physics/pumas/micro_mg3_0.F90': No such file or directory
#18 ERROR: process "/bin/sh -c git clone ... rm -rf /containers" did not complete successfully: exit code: 1
...
Dockerfile:68
  79 | >>> cp -rf /containers/CESM/2.2/Files/micro_mg3_0.F90 /opt/ncar/cesm2/components/cam/src/physics/pumas/micro_mg3_0.F90 && \
Finished: FAILURE
```

### 根因定位
- 失败位置: `HPC/cesm/2.1.5/24.03-lts-sp4/Dockerfile:79`（新增文件第 79 行，即 `RUN git clone ... && cp -rf ...` 复合步骤中的 `micro_mg3_0.F90` 拷贝行）
- 失败原因: `cp -rf` 的源是单个文件，而目标路径的父目录 `/opt/ncar/cesm2/components/cam/src/physics/pumas/` 在刚 clone 下来的 CESM **2.1.5** 源码树中不存在，`cp` 不会自动创建父目录，因此报 `No such file or directory`，整条 RUN 以 exit code 1 失败。

### 与 PR 变更的关联
本次 PR 新增了 `HPC/cesm/2.1.5/24.03-lts-sp4/Dockerfile`（88 行全部新增），失败发生在该新文件的第 68–82 行步骤中。

关键矛盾点：
- `ARG VERSION=2.1.5`，`git clone -b release-cesm${VERSION}` 实际拉取的是 **CESM 2.1.5** 源码（日志 `#16` 已确认 clone/checkout 成功）；
- 但所有配置文件来源被硬编码为 **`/containers/CESM/2.2/Files/`**（ESCOMP-Containers 的 CESM 2.2 目录），且拷贝目标路径（`components/cam/src/physics/pumas/`）也是按 CESM 2.2 的目录结构书写的。

`pumas` 是 CESM 2.2 中 CAM 物理模块引入的新目录结构，CESM 2.1.5 中并不存在该路径。因此把 2.2 的容器文件路径套用到 2.1.5 源码树上，必然导致目标父目录缺失。该 Dockerfile 很可能由更高版本（2.2.x）的 Dockerfile 复制而来，仅把版本号改为 2.1.5，但未同步调整上游 `Files` 路径与目标路径。

补充说明（非根因）：
- `#16 14.37 warning: refs/tags/release-cesm2.1.5 ... is not a commit!` 只是 git 警告，clone 与 `checkout_externals` 均成功（`#16 DONE`）。
- 前面的 mpich/hdf5/netcdf/pnetcdf 编译安装均成功（`#14 DONE`、`PnetCDF has been successfully installed`），与本次失败无关。

## 修复方向

### 方向 1（置信度: 高）
统一版本基线：将配置文件的来源目录由 `/containers/CESM/2.2/Files/` 改为与 `VERSION=2.1.5` 对应的上游目录（ESCOMP-Containers 中对应 2.1.5 的 `Files` 目录）。这样源文件与目标路径均按 2.1.5 的目录结构书写，`pumas` 等不存在的路径问题自然消除。需确认上游 ESCOMP-Containers 中确实存在该版本目录，且其文件清单与本 Dockerfile 逐条对应。

### 方向 2（置信度: 中）
若上游 ESCOMP-Containers 没有 2.1.5 目录、只能沿用 2.2 的 `Files`，则需按 CESM 2.1.5 的实际源码结构逐一修正拷贝目标路径，而非沿用 2.2 结构：
- `micro_mg3_0.F90` 的落地目录在 2.1.5 下应指向 CAM 实际的 physics 目录（2.2 才迁移到 `physics/pumas/`）；
- `scam_shell_commands` 的 `usermods_dirs/scam_mandatory/` 与 `create_scam6_iop` 的 `components/cam/bld/scripts/` 等目标目录也需确认在 2.1.5 树中存在；
- 对确需保留的 2.2 结构目标，应在 `cp` 前显式创建父目录。

方向 2 风险较高（2.2 的源码/配置文件与 2.1.5 不兼容），仅在上游确无对应版本目录时采用。

## 需要进一步确认的点
1. ESCOMP-Containers 仓库中是否存在与 2.1.5 对应的 `Files` 目录及其文件清单（决定采用方向 1 还是方向 2）。
2. CESM 2.1.5 源码树中 `components/cam/src/physics/` 下 `micro_mg3_0.F90` 的真实相对路径（确认 2.1.5 是否已存在 `pumas/` 目录）。
3. 第 68–82 行其余 `cp` 目标目录（`cime/config/cesm/machines/`、`cime_config/`、`components/*/cime_config/`、`usermods_dirs/scam_mandatory/`、`bld/scripts/`）在 CESM 2.1.5 中是否全部存在；日志仅显示 `pumas` 一处报错，但需确认后续路径不会连锁失败。
4. 该 Dockerfile 是否与仓库中既有的 `HPC/cesm/2.2.x` 版本高度雷同（可佐证“从高版本复制、仅改版本号”的推断），并据此决定正确基线。

## 修复验证要求（涉及引用外部源文件）
本修复需修改对上游 `ESCOMP-Containers` 仓库文件的引用路径，code-fixer 在提交前必须：
1. 依据 Dockerfile 中的 `ARG VERSION=2.1.5`，从 `https://github.com/ESCOMP/ESCOMP-Containers` 确认对应版本目录是否存在及其准确路径，验证新引用的 `.../Files/...` 源文件路径确实存在。
2. 从 `https://github.com/ESCOMP/cesm` 的 `release-cesm2.1.5` 分支确认 `components/cam/src/physics/` 下 `micro_mg3_0.F90` 的实际路径，验证新目标路径的父目录确实存在。
3. 逐条核对第 68–82 行所有 `cp` 的源/目标，确保修改后不再有父目录缺失的路径。

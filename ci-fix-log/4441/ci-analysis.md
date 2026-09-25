# CI 失败分析报告

## 基本信息
- PR: #4441 — 【自动升级】cesm容器镜像升级至2.1.5版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: CESM配置版本不匹配
- 新模式症状关键词: cannot create regular file, No such file or directory, cp, micro_mg3_0.F90, pumas, ESCOMP-Containers

## 根因分析

### 直接错误
```
#18 0.066 Cloning into '/containers'...
#18 0.963 cp: cannot create regular file '/opt/ncar/cesm2/components/cam/src/physics/pumas/micro_mg3_0.F90': No such file or directory
#18 ERROR: process "/bin/sh -c git clone https://github.com/ESCOMP/ESCOMP-Containers.git /containers && ... cp -rf /containers/CESM/2.2/Files/micro_mg3_0.F90 /opt/ncar/cesm2/components/cam/src/physics/pumas/micro_mg3_0.F90 ..." did not complete successfully: exit code: 1
```
（报错为 `cannot create regular file`，即**目标路径的父目录不存在**，而非源文件缺失；若是源文件缺失，报错应为 `cannot stat`。）

### 根因定位
- 失败位置: `HPC/cesm/2.1.5/24.03-lts-sp4/Dockerfile:79`
- 失败原因: 新 Dockerfile 克隆的是 CESM `release-cesm2.1.5` 源码，却从 `ESCOMP-Containers` 仓库的 **CESM 2.2** 目录（`/containers/CESM/2.2/Files/…`，硬编码）复制配置与源码文件；CESM 2.1.5 的 CAM 源码树中不存在 CESM 2.2 才引入的 `components/cam/src/physics/pumas/` 目录，导致 `cp` 无法写入该路径而失败。

### 与 PR 变更的关联
本次失败由 PR 新增的 `HPC/cesm/2.1.5/24.03-lts-sp4/Dockerfile` 直接引起：
- 该文件的 `ARG VERSION=2.1.5`，`git clone -b release-cesm2.1.5`（日志 `#16` 已成功克隆并完成 `checkout_externals`，`#16 DONE 126.8s`）。
- 但同文件第 68–82 行仍使用上一版本模板中的固定路径 `/containers/CESM/2.2/Files/...`，并将 `micro_mg3_0.F90` 复制到 2.1.5 源码并不存在的 `components/cam/src/physics/pumas/`。
- 前序若干 `cp`（cime/config、cice、cism、pop、clm 的 cime_config 等）目标目录在 2.1.5 中恰好存在，因此直到 CAM 物理源码目录这一条才暴露版本路径不匹配。
- 日志中 `#16 warning: refs/tags/release-cesm2.1.5 … is not a commit!` 为 tag 解引用提示，git 已成功切换到对应 commit，非本次失败原因；`#13`/`#14`（mpich/hdf5/netcdf/pnetcdf 等依赖）均已 `DONE`，与失败无关。

## 修复方向

### 方向 1（置信度: 高）
将「容器配置来源版本」与「克隆的 CESM 版本」对齐：把 `/containers/CESM/2.2/Files/...` 的硬编码版本目录改为与 `VERSION` 一致的 CESM 版本目录（使用 `${VERSION}` 或对应版本号），并确认 `ESCOMP-Containers` 中确实存在该版本目录。即二选一：或用 CESM 2.2 源码 + 2.2 配置，或用 CESM 2.1.5 源码 + 2.1.5 对应配置。

### 方向 2（置信度: 中）
若确认 `ESCOMP-Containers` 只有 CESM 2.2 的 Files，则应调整复制目标路径以适配 CESM 2.1.5 的实际目录结构（例如 CESM 2.1.5 的 CAM 物理目录并非 `src/physics/pumas`），并为 2.1.5 提供对应版本的配置文件，而非直接沿用 2.2 模板。

## 需要进一步确认的点
- 确认 `ESCOMP-Containers` 仓库中是否存在与 2.1.5 对应的目录（如 `CESM/2.1`），若无，2.1.5 应使用哪一套 Files。
- 确认 CESM `release-cesm2.1.5` 源码中 CAM 物理源码的实际路径（`components/cam/src/physics/` 下的子目录名），以确定 `micro_mg3_0.F90` 的正确目标路径。
- 确认该升级 PR 的基线版本（原 Dockerfile 所在的 CESM 版本）及目标产物版本，判断应以源码版本还是配置版本为准进行对齐。

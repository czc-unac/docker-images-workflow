# CI 失败分析报告

## 基本信息
- PR: #4525 — 【自动升级】cesm容器镜像升级至2.1.5版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式（与模式12「上游代码目录结构变更」相关但不完全等同）
- 新模式标题: 跨版本配置路径缺失
- 新模式症状关键词: `cp: cannot create regular file`, `No such file or directory`, `pumas`, `micro_mg3_0.F90`, `ESCOMP-Containers`, `CESM/2.2`

## 根因分析

### 直接错误
```
#18 0.058 Cloning into '/containers'...
#18 0.972 cp: cannot create regular file '/opt/ncar/cesm2/components/cam/src/physics/pumas/micro_mg3_0.F90': No such file or directory
#18 ERROR: process "/bin/sh -c git clone https://github.com/ESCOMP/ESCOMP-Containers.git /containers && ... cp -rf /containers/CESM/2.2/Files/micro_mg3_0.F90 /opt/ncar/cesm2/components/cam/src/physics/pumas/micro_mg3_0.F90 && ... did not complete successfully: exit code: 1

------
Dockerfile:68
  79 | >>>     cp -rf /containers/CESM/2.2/Files/micro_mg3_0.F90 /opt/ncar/cesm2/components/cam/src/physics/pumas/micro_mg3_0.F90 && \
ERROR: failed to solve: process "... did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `HPC/cesm/2.1.5/24.03-lts-sp4/Dockerfile:79`（[12/13] 步骤内，行号对应 Dockerfile:68~82 的 RUN 块）
- 失败原因: 该 RUN 步骤把 **ESCOMP-Containers 仓库中 `CESM/2.2/` 版本目录**下的配置文件，复制到 **CESM 2.1.5** 的源码树中；目标路径 `/opt/ncar/cesm2/components/cam/src/physics/pumas/` 在 CESM 2.1.5 中不存在（`pumas` 目录属于 CESM 2.2 的 CAM 物理目录结构），`cp` 因目标父目录缺失报 `cannot create regular file ... No such file or directory`。

### 与 PR 变更的关联
本 PR 新增了 `HPC/cesm/2.1.5/24.03-lts-sp4/Dockerfile`（a_mode=0，new_file），整条失败链路均在该新增文件内。`VERSION=2.1.5` 克隆的是 `release-cesm2.1.5`，但配置来源目录硬编码为 `ESCOMP-Containers` 的 `CESM/2.2/Files/`（Dockerfile:68~82），版本不匹配直接导致复制失败，属本次 PR 改动引入。

## 修复方向

### 方向 1（置信度: 高）
将配置来源从 `ESCOMP-Containers` 的 `CESM/2.2/Files/` 调整为与 2.1.5 匹配的目录，并核对每个 `cp` 的目标路径在 CESM 2.1.5 源码树中确实存在；对确实不存在对应目录的条目（如 `micro_mg3_0.F90` 的目标 `physics/pumas/`），需按 2.1.5 的实际目录布局重定向到正确位置。

### 方向 2（置信度: 中）
若上游 `ESCOMP-Containers` 确有适用于 2.1.5 的 Files 集合，则整体替换配置源版本；否则需要针对 2.1.5 单独确定配置集，并对缺失的目标目录做合理处理（如创建目录或改用 2.1.5 实际存在的路径），不能沿用 2.2 的目录假设。

## 需要进一步确认的点
- `ESCOMP-Containers` 仓库是否存在 `CESM/2.1.5/Files/`（或其它适配 2.1.5 的配置目录）；若无，应使用哪一套配置。
- CESM `release-cesm2.1.5` 源码树中 `micro_mg3_0.F90` 的真实路径（2.1.x 中 CAM 物理文件通常不在 `physics/pumas/` 下）。
- 该 RUN 中其余 `cp` 目标（`config_machines.xml`、`config_pes.xml`、各 `configs/*/config_pes.xml` 等）在 2.1.5 下是否同样依赖 2.2 的目录布局，可能仍有后续失败点。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
本次不涉及对第三方源文件的正则 patch。但 `code-fixer` 在提交前必须验证：所选配置源目录与 `ARG VERSION=2.1.5`/`release-cesm2.1.5` 匹配，且 Dockerfile 中每个 `cp` 的目标目录在 CESM 2.1.5 源码树中真实存在（以 `release-cesm2.1.5` 拉取结果为准），不能沿用 2.2 的路径假设。

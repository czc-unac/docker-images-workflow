# 修复摘要

## 修复的问题
修复 `HPC/cesm/2.1.5/24.03-lts-sp4/Dockerfile` 中错误地套用 CESM 2.2 的配置文件路径，导致 `cp` 目标父目录 `components/cam/src/physics/pumas/` 在 CESM 2.1.5 源码树中不存在、构建以 exit code 1 失败的问题。

## 修改的文件
- `HPC/cesm/2.1.5/24.03-lts-sp4/Dockerfile`: 将 `# Get config files` 步骤的配置文件来源由 `/containers/CESM/2.2/Files/` 改为与 `ARG VERSION=2.1.5` 对应的 `/containers/CESM/2.1/Files/`，并只保留 CESM 2.1 实际存在且需要的 3 个文件（`config_machines.xml`、`config_inputdata.xml`、`case_setup.py`），删除仅属于 CESM 2.2 的 9 条 `cp`（`config_compsets.xml`、`config_pes.xml`、`configs/*/config_pes.xml`、`micro_mg3_0.F90`、`scam_shell_commands`、`create_scam6_iop`）。

## 修复逻辑
- 根因：本 Dockerfile 是从仓库既有的 `HPC/cesm/2.2.2/24.03-lts-sp4/Dockerfile` 复制而来（两者除 `VERSION` 外逐字节相同），仅把版本号改为 2.1.5，却仍引用上游 `ESCOMP-Containers` 的 `CESM/2.2/Files` 与 CESM 2.2 的源码目录结构。`pumas` 是 CESM 2.2 才引入的 CAM 新目录，CESM 2.1.5 中不存在，故 `cp -rf` 单个文件到该目录时报 `No such file or directory`，整条 RUN 失败。
- 修复方向采用分析报告的“方向 1（统一版本基线）”，并以 ESCOMP 官方 `CESM/2.1/Dockerfile` 为权威基线。已从上游实际验证：
  1. `https://github.com/ESCOMP/ESCOMP-Containers`（`main`）中 `CESM/2.1/Files/` 仅包含 `case_setup.py`、`config_compilers.xml`、`config_inputdata.xml`、`config_machines.xml`、`ea002e626aee6bc6643e8ab5f998e5e4`，**不存在** `config_compsets.xml`、`config_pes.xml`、`configs/`、`micro_mg3_0.F90`、`scam_shell_commands`、`create_scam6_iop`；`CESM/2.2/Files/` 才包含这些 2.2 专有文件。故 2.2 的 `cp` 源必须删除而非改路径。
  2. 上游 `ESCOMP-Containers` 的 `CESM/2.1/Dockerfile` 对 2.1 只拷贝 `config_compilers.xml`、`config_machines.xml`、`config_inputdata.xml`、`case_setup.py` 四个文件，与本修复一致；其中 `config_compilers.xml` 仍由本仓库本地 `COPY config_compilers.xml ...`（第 74 行）提供，保持不变。
  3. 依据 `CESM` 仓库 `release-cesm2.1.5` 的 `Externals.cfg`，其 `cime` 外部版本为 `cime5.6.49`（`ESMCI/cime`），已确认目标目录/文件真实存在：`cime/config/cesm/machines/`、`cime/config/cesm/` 均存在，`cime/scripts/lib/CIME/case/case_setup.py` 返回 HTTP 200。
  4. 其 CAM 外部版本为 `cam_cesm2_1_rel_60`（`ESCOMP/CAM`），已确认 `src/physics/pumas/` 与 `micro_mg3_0.F90` **均不存在**（CAM 2.1 只有 `src/physics/cam/micro_mg1_0/2_0.F90` 等），进一步证明删除 `micro_mg3_0.F90` 拷贝是正确且必要的。
- 结论：修复后所有 `cp` 的源文件路径均真实存在，目标父目录均真实存在，不会再出现父目录缺失错误。

## 潜在风险
- 本次改动移除了 CESM 2.2 专有的 SCAM 修复（`micro_mg3_0.F90`、`scam_shell_commands`、`create_scam6_iop`）及各组件 `config_pes.xml`/`config_compsets.xml`。这与上游 CESM 2.1 官方镜像行为一致，且 2.1.5 中不存在 `pumas`、`micro_mg3_0.F90`，这些文件本就不适用于 2.1.5；但如果后续有基于 SCAM / 特定 compset 的验证需求，需按 CESM 2.1 的实际文件另行处理。
- 本地 `config_compilers.xml` 与仓库内 2.2.2 版本逐字节相同（并非上游 2.1 版本），属于 openEuler 既有定制，未在本次 CI 失败范围内，按最小化原则未改动；若后续运行时报编译器配置不兼容，可再单独评估。
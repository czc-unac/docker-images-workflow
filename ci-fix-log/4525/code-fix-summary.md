# 修复摘要

## 修复的问题
CESM 2.1.5 镜像构建时错误地复制 CESM 2.2 的容器配置文件（包括 2.2 才存在的 `physics/pumas/micro_mg3_0.F90`），导致 `cp: cannot create regular file ... No such file or directory` 构建失败。

## 修改的文件
- `HPC/cesm/2.1.5/24.03-lts-sp4/Dockerfile`: 将配置来源由 `ESCOMP-Containers/CESM/2.2/Files/` 改为与 CESM 2.1.x 匹配的 `ESCOMP-Containers/CESM/2.1/Files/`，仅保留该版本目录中真实存在的 3 个文件（`config_machines.xml`、`config_inputdata.xml`、`case_setup.py`），删除 2.2 专有且 2.1 源码树中不存在目标路径的 10 条 `cp`（`config_compsets.xml`、`config_pes.xml`、各 `configs/*/config_pes.xml`、`micro_mg3_0.F90`、`scam_shell_commands`、`create_scam6_iop`）。

## 修复逻辑
CI 失败根因（分析报告方向 2：上游存在适配 2.1 的配置集）：本 PR 新增的 Dockerfile 中 `ARG VERSION=2.1.5` 克隆的是 `release-cesm2.1.5`，但配置来源硬编码为 `ESCOMP-Containers` 的 `CESM/2.2/Files/`。经查上游 `ESCOMP/ESCOMP-Containers` 仓库，实际只存在 `CESM/2.1/` 与 `CESM/2.2/` 两个版本目录，其中 `CESM/2.1/Files/` 恰好只包含 `case_setup.py`、`config_compilers.xml`、`config_inputdata.xml`、`config_machines.xml` 四个文件——这正是上游 `CESM/2.1/Dockerfile` 为 CESM 2.1.x 使用的配置集。

因此按上游 `CESM/2.1/Dockerfile` 的做法进行最小化对齐：仅复制 `CESM/2.1/Files/` 下真实存在的 3 个文件（`config_compilers.xml` 已由本 PR 的本地文件在第 74 行 COPY 覆盖，故无需从上游取）。其余 10 个文件均属 CESM 2.2 专有：

- `config_compsets.xml`、`config_pes.xml`、`configs/*/config_pes.xml` 为 2.2 的 XML 配置改动，2.1 系列不存在且上游 2.1 容器并不需要；
- `micro_mg3_0.F90` 目标路径 `components/cam/src/physics/pumas/` 属于 CAM 2.2 的物理目录结构，在 2.1 中不存在（分析报告已指出），上游 2.1 容器也未使用该文件；
- `scam_shell_commands`、`create_scam6_iop` 同样为 2.2 的 SCAM 修复，2.1 源码树缺少对应的目标父目录。

## 验证结果（源文件与目标路径核对）
已通过网络获取上游实际内容并逐一验证：

1. **配置源目录核实**：`https://api.github.com/repos/ESCOMP/ESCOMP-Containers/contents/CESM` 返回目录仅为 `2.1`、`2.2`（无 `2.1.5`）；递归 tree 显示 `CESM/2.1/Files/` 实际文件为 `case_setup.py`、`config_compilers.xml`、`config_inputdata.xml`、`config_machines.xml`、`ea002e626aee6bc6643e8ab5f998e5e4`。本次使用的 3 个源文件均确认存在。
2. **上游参考 Dockerfile**：`https://raw.githubusercontent.com/ESCOMP/ESCOMP-Containers/main/CESM/2.1/Dockerfile` 对 `release-cesm2.1.3` 使用的目标路径与本修复完全一致。
3. **目标路径核实（按精确 tag `release-cesm2.1.5`）**：该 tag 存在（tag object `258eca2fda1b2487c4c97b9e5f8643d3e3f84142`）。其 `Externals.cfg` 指定 cime external 为 `ESMCI/cime` tag `cime5.6.49`；在 `cime5.6.49` 中确认：`config/cesm/machines/` 存在、`config/cesm/` 存在（含 `config_inputdata.xml`）、`scripts/lib/CIME/case/case_setup.py` 存在（HTTP 200）。三条 `cp` 目标父路径均真实存在。
4. `release-cesm2.1.5` 在 ESCOMP/cesm 的 tags 列表中确认存在，`git clone -b release-cesm2.1.5` 有效。

本次不涉及对第三方源文件的正则 patch，无正则需验证。

## 潜在风险
- `CESM/2.1/Files` 配置集对应当前上游 2.1 系列（上游示例为 `release-cesm2.1.3`），与 `release-cesm2.1.5` 同属 2.1.x，目录与接口布局一致，兼容性风险低。
- 删除了 2.2 的 `config_pes.xml` / SCAM 相关修复：这些文件在 CESM 2.1 中原本无法落地（目标目录不存在），删除后与上游 2.1 容器行为一致；不会影响基础镜像构建。若后续 README 中的 `F2000climo` 测试对 PE 布局有额外要求，需另行按 2.1 的 `config_pes.xml` 单独适配，但不属于本次 CI 失败根因范围。
- 未修改 `config_compilers.xml`（本 PR 已包含的 `container` MACH 配置），与 2.2.2 版本一致且兼容 2.1 的 `config_compilers version="2.0"` 格式。
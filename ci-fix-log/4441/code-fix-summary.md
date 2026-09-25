# 修复摘要

## 修复的问题
CESM 2.1.5 Dockerfile 误用了 `ESCOMP-Containers` 仓库中 CESM 2.2 的配置/源码文件（硬编码 `/containers/CESM/2.2/Files/...`），导致构建时 `cp` 到 2.1.5 源码树中不存在的路径而失败。

## 修改的文件
- `HPC/cesm/2.1.5/24.03-lts-sp4/Dockerfile`: 将配置来源目录由 `/containers/CESM/2.2/Files/` 改为与 `VERSION=2.1.5` 对应的 `/containers/CESM/2.1/Files/`；同时删除 `CESM/2.1/Files` 中并不存在、仅 CESM 2.2 才提供的 `cp` 条目（`config_compsets.xml`、`config_pes.xml`、`configs/{cam,cice,cism,pop,clm}/config_pes.xml`、`micro_mg3_0.F90`、`scam_shell_commands`、`create_scam6_iop`）。

## 修复逻辑
- 分析报告根因：失败位于 `Dockerfile:79`，`cp: cannot create regular file '.../components/cam/src/physics/pumas/micro_mg3_0.F90': No such file or directory`。原因是克隆的源码为 `release-cesm2.1.5`，却从 `ESCOMP-Containers` 的 **CESM 2.2** 目录复制配置与源码，CESM 2.1.5 中不存在 2.2 才引入的 `components/cam/src/physics/pumas/` 目录。
- 已从上游验证：`https://github.com/ESCOMP/ESCOMP-Containers`（默认分支 main）下确实存在 `CESM/2.1` 与 `CESM/2.2` 两个目录，其中 `CESM/2.1/Files` 仅包含 `case_setup.py`、`config_compilers.xml`、`config_inputdata.xml`、`config_machines.xml`；`config_compsets.xml`、`config_pes.xml`、`configs/`、`micro_mg3_0.F90`、`scam_shell_commands`、`create_scam6_iop` 均只存在于 `CESM/2.2/Files`。
- 修复与上游 `ESCOMP-Containers/CESM/2.1/Dockerfile` 的做法保持一致：2.1 容器只复制 `config_machines.xml`、`config_inputdata.xml`、`case_setup.py`（以及 `config_compilers.xml`）。因此将来源统一改为 `CESM/2.1`，并移除 2.1 不提供的文件复制，既修复了 CI 失败，也避免了改用 2.1 源后出现的 `cannot stat` 新错误。
- 已确认保留的 3 个 `cp` 源文件在 `CESM/2.1/Files` 中存在，且目标父目录（`cime/config/cesm/machines/`、`cime/config/cesm/`、`cime/scripts/lib/CIME/case/`）由 CESM 2.1.5 的 `checkout_externals` 生成，复制可成功。
- 未改动 `config_compilers.xml`：经比对，其 gnu 编译器段与上游 2.1/2.2 版本功能一致（仅 `compile_threaded` 大小写差异），2.1.5 下可正常使用，且 Docker 构建阶段不会因其内容失败，按最小化原则不修改。

## 潜在风险
无。改动仅限 2.1.5 目录的 Dockerfile 配置复制逻辑，不影响 2.2.2 等其他版本；其余文件（README.md、image-info.yml、meta.yml）无需修改。
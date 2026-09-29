# 修复摘要

## 修复的问题
无需修改代码。CI 失败分析报告将本次失败判定为 `infra-error`（证据不足，`ci.logs` 与 `ci.run_info` 均缺失），无法定位真实根因，故未做任何代码改动。

## 修改的文件
- 无（未修改任何文件）

## 修复逻辑
分析报告明确指出：失败类型为 `infra-error`，置信度低，缺少失败 job 的完整构建日志，无法区分是编译失败、依赖问题、启动 check 还是预检/编排阶段问题。按任务约定，`infra-error` 不应强行修改代码。

为排除报告"方向 2"中列出的 5 个 diff 级静态风险点，已在源码库中逐一核实，结论如下：

1. **许可证头缺失（疑似 `check_package_license`）— 已排除**：
   同目录既有构建文件 `HPC/cp2k/2024.3/24.03-lts/Dockerfile`、`HPC/cp2k/2025.2/24.03-lts-sp2/Dockerfile`、`HPC/cp2k/2025.2/24.03-lts-sp4/Dockerfile` 同样首行为 `ARG BASE=...`，均无 Copyright/SPDX 头。既有文件可正常构建/合并，说明该检查并未对本路径强制要求许可头，属于误报，不应新增头（否则与同类文件风格不一致）。

2. **上游 `support/v2026.2` 分支不存在 — 已排除**：
   通过 `git ls-remote --heads https://github.com/cp2k/cp2k.git` 实际查询，确认上游存在 `refs/heads/support/v2026.2`（commit 67b5da876dd6a76b8b021d5a04d1c81ba79a4c50），`git clone -b support/v2026.2` 可正常解析。

3. **工具链安装脚本参数不兼容 — 已排除**：
   已从上游按实际 tag/branch 拉取 `support/v2026.2/tools/toolchain/install_cp2k_toolchain.sh`（1609 行）并核对。Dockerfile 中使用的全部选项（`--install-all`、`--enable-cuda=no`、`--with-deepmd`、`--target-cpu`、`--with-cusolvermp`、`--with-libtorch`、`--with-gcc`、`--with-openmpi`、`--with-sirius`、`--with-elpa`、`--with-plumed`、`--with-libvori`、`--with-cosma`、`--with-libsmeagol`、`--with-dftd4`、`--with-tblite`、`--with-trexio`、`--with-greenx`、`--with-gmp`、`--with-spfft`、`--with-spla`、`--with-gsl`、`--with-spglib`、`--with-hdf5`、`--with-libvdwxc`）在 2026.2 脚本的参数解析分支（约 793–975 行）及 `--help` 说明中均有定义，未被移除，不会触发 `Unknown flag`。

4. **`HPC/image-list.yml` 未登记 cp2k — 已排除**：
   `HPC/image-list.yml` 第 5 行已存在 `cp2k: cp2k` 条目，无需新增；且该文件不在 `pr.changed_files` 允许修改范围内。

5. **文件末尾无换行 — 非构建失败因素**：
   同类既有 Dockerfile/文档同样如此，属仓库既有格式，不影响 Docker 构建或启动 check。

此外，对比 2026.2 与 2025.2 的 Dockerfile，二者仅有 `ARG VERSION`（2025.2→2026.2）与安装包列表新增 `xz` 两处差异，改动合理且最小（`xz` 为解压依赖）。新增的 README.md / doc/image-info.yml / meta.yml 内容与版本信息一致，tag 表、meta.yml 路径条目均已正确补充。

综上，本次 PR 的四类改动均无明显缺陷，现有证据不足以支撑任何代码级修复。建议补充失败 job 的完整 `ci.logs`（含最早 ERROR / exit code 行、失败阶段、失败架构）后重新分析，而非在缺少日志的情况下盲改 Dockerfile。

## 潜在风险
无代码改动，无引入新风险。若后续拿到真实日志确认属于上游 cp2k 2026.2 工具链行为变更（例如某默认依赖在 `--install-all` 下被要求联网或编译失败），需再单独定位并修复。
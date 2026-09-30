# 修复摘要

## 修复的问题
cp2k 2026.2 构建阶段缺少 `xz` 工具，导致 toolchain 的 `install_libint.sh` 解压 `libint-*.tar.xz` 失败（x86_64 与 aarch64 均失败）。

## 修改的文件
- `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile`: 在 build 阶段的 `yum install` 列表中新增 `xz` 包（`... openssh-clients bzip2 xz \`）。

## 修复逻辑
1. 分析报告因日志缺失给出低置信度推断，本人进一步获取了真实 CI 日志：
   - PR #4763 的门禁结果表格中，`check_package_license` 仅为 WARNING（仓库级 copyright 文件缺失），`check_sca` SUCCESS，真正的失败项是 x86_64 与 aarch64 的 `check_build`（均为 FAILED）。
   - 通过 openEuler log-ci 日志接口拉取构建日志：
     - x86_64: `.../x86-64/openeuler-docker-images/4874/`
     - aarch64: `.../aarch64/openeuler-docker-images/4970/`
   - 两个架构的第一条真实错误一致：
     ```
     #11 tar (child): xz: Cannot exec: No such file or directory
     #11 tar (child): Error is not recoverable: exiting now
     #11 ERROR: (./scripts/stage3/install_libint.sh, line 53) Non-zero exit code detected.
     ```
2. 根因确认：从上游 cp2k `support/v2026.2` 获取 `tools/toolchain/scripts/stage3/install_libint.sh` 验证，该版本将 libint 包改为 `libint-v2.13.1-cp2k-lmax-*.tar.xz`，并使用 `tar -xJf`（xz 解压）；而 2025.2 版本使用 `tar -xzf`（gzip）。openEuler 24.03-LTS-SP4 基础镜像未预装 `/usr/bin/xz`，因此解压失败。
3. 修复：在 build 阶段安装 `xz` 包（openEuler 中该包提供 `xz` 命令，已用仓库内其它 Dockerfile 的 `yum/dnf install xz` 用法佐证包名）。仅改动构建依赖，未触碰无关代码。

## 潜在风险
无。仅在构建阶段新增一个系统包，不影响运行时镜像内容与已有构建逻辑；运行时阶段不需要 xz。
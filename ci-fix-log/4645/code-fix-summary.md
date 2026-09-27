# 修复摘要

## 修复的问题
CP2K 2026.2 构建阶段缺少 `xz` 解压工具，导致工具链安装 LIBINT 时无法解压 `libint-v2.13.1-cp2k-lmax-5.tar.xz` 而失败。

## 修改的文件
- `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile`: 在构建阶段第一个 `yum install -y ...` 的包列表（第 7 行）中，于 `bzip2` 后补充 `xz` 包。

## 修复逻辑
CI 日志显示 `tar (child): xz: Cannot exec: No such file or directory`，失败点为上游 `tools/toolchain/scripts/stage3/install_libint.sh:53`，根因是容器内缺少 `/usr/bin/xz`，`tar` 无法处理 `.tar.xz` 格式的 libint 源码包。原 Dockerfile 的依赖清单中已含 `bzip2`（对应 `.tar.bz2`）却遗漏 `xz`，而 CP2K 2026.2 工具链的 LIBINT 包改用 `.tar.xz`，因此在 LIBINT 阶段才暴露。修复对应分析报告的「方向 1（置信度: 高）」：向 openEuler 24.03-LTS-SP4 基础镜像的包列表补充 `xz`（该包提供 `xz` 可执行文件）。改动仅一行，未触碰无关文件。此为 Dockerfile 系统包安装列表变更，不涉及对第三方/上游源文件的正则 patch，无需上游正则验证。

## 潜在风险
- 需实际触发一次 Docker 构建验证：确认 LIBINT 及后续工具链步骤可正常解压 `.tar.xz` 并继续；分析报告提示后续被禁用的可选包若解禁可能仍存在其他缺失依赖，但当前构建参数已禁用，暂不影响。
- openEuler 24.03-LTS-SP4 中提供 `xz` 可执行文件的准确 RPM 包名以 `xz` 为准（同 RHEL/openEuler 惯例），若构建环境元数据有差异，可回退核实为 `xz-libs`/`xz-devel`，但通常 `xz` 即提供 `/usr/bin/xz`。
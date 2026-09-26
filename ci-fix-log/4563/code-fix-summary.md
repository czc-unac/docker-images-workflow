# 修复摘要

## 修复的问题
CP2K 2026.2 构建阶段缺少 xz 解压工具，导致工具链安装 LIBINT 时解压 `.tar.xz` 失败。

## 修改的文件
- `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile`: 在 build 阶段 `yum install -y` 依赖列表中加入 `xz`。

## 修复逻辑
分析报告根因明确：build 阶段安装的包列表遗漏 `xz`，而 CP2K 2026.2 工具链 `scripts/stage3/install_libint.sh` 需要解压 `libint-v2.13.1-cp2k-lmax-5.tar.xz`，`tar` 调用外部 `xz` 解压器时报 `xz: Cannot exec: No such file or directory` 并中断安装。日志显示下载与校验均成功（OK/Checksum Ok），且前面的 `.tar.bz2` 文件（bzip2 已安装）解压正常，进一步印证是缺少 xz 工具。在 build 阶段依赖列表中补充 `xz` 后，`tar` 即可正常解压 xz 格式包，`install_cp2k_toolchain.sh` 得以继续执行。改动仅一行、范围最小，不影响运行阶段（运行阶段不解压源码包，entrypoint 仅 source 环境变量）。

## 潜在风险
- 日志在 LIBINT 阶段即中断，2026.2 工具链后续阶段（libxc/gauxc/libxsmm 等）是否还存在新增依赖缺失无法覆盖，需重新触发完整构建验证。
- 若后续编译需要 liblzma 头文件（xz-devel），可能需进一步补充；当前仅运行期解压需求，暂只添加 `xz`。
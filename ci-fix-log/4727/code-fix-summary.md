# 修复摘要

## 修复的问题
修复 cp2k 2026.2 镜像构建失败：上游 CP2K 2026.2 已移除 Makefile 构建体系（不再生成 `install/arch/local.psmp`），Dockerfile 仍执行 `make ARCH=local VERSION=psmp`，导致 `cp: cannot stat './tools/toolchain/install/arch/local.psmp'` + `make: *** No targets specified and no makefile found.`，两个架构（x86-64 / aarch64）均在 `[build 7/8]` 步骤以 exit code 2 失败。

## 修改的文件
- `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile`:
  - 构建步骤由 Makefile 改为上游 2026.2 官方 CMake 构建脚本：`WORKDIR /opt/cp2k/tools/toolchain` + `./build_cp2k.sh --build-static -j $(nproc)`（该脚本要求 cwd 为 `tools/toolchain`，会自动读取 `toolchain.conf`/`toolchain_settings` 生成 CMake 选项）。
  - 运行时依赖收集步骤改为对 `/opt/cp2k/install/bin/cp2k.psmp` 执行 `ldd`，并先 `source .../install/setup` 使 ldd 能解析工具链共享库；移除已不存在的 `unlink ./exe/local/cp2k.popt`、`unlink ./exe/local/cp2k_shell.psmp`。
  - runtime 阶段由 `COPY /opt/cp2k/exe/local/` 改为 `COPY /opt/cp2k/install/`（CMake 的安装前缀布局）。
  - 软链接目标与 entrypoint 的 `PATH` 由 `/opt/cp2k/exe/local` 更新为 `/opt/cp2k/install/bin`。
  - `run_tests` 调用按 2026.2 的 `do_regtest.py <binary_dir> <version>` 新签名，把 `local` 改为 `/opt/cp2k/install/bin`。
  - 在 `--install-all` 基础上补充 `--with-ace=no --with-gauxc=no`：`gauxc` 会强制启用预编译 `libtorch`（仅 x86_64），在 aarch64 上链接会失败；上游 2026.2 的 arm64 构建 profile（`toolchain_arm64`）正是关闭 ACE/LIBTORCH/GAUXC。原 Dockerfile 已有 `--with-libtorch=no`，本次补齐 `--with-gauxc=no` 后 libtorch 才会保持关闭。

## 修复逻辑
已获取真实失败 job 日志并定位根因：
- x86-64 构建日志：`#13 0.053 cp: cannot stat './tools/toolchain/install/arch/local.psmp': No such file or directory` + `#13 0.181 make: *** No targets specified and no makefile found. Stop.`（同样出现在 aarch64 日志）。
- 通过上游 `cp2k/cp2k` `support/v2026.2`（对应 Dockerfile `ARG VERSION=2026.2`）验证：`tools/toolchain/generate_arch_files.sh` 在 2026.2 已被删除（2025.2 仍存在），toolchain 结束时提示改用 `./build_cp2k.sh`；新脚本基于 CMake，二进制安装到 `CMAKE_INSTALL_PREFIX=/opt/cp2k/install/bin`（`cp2k.psmp`、`cp2k.popt` 软链、`cpk_shell`/`graph`/`dumpdcd`/`xyz2dcd` 的 `.psmp` 均在 `install/bin`），并生成 `install/cp2k_env`。
- 因此把构建方式切换到 `build_cp2k.sh --build-static`（静态 `libcp2k` 与旧 `make` 产物等价，外部依赖仍为共享库，`ldd` 收集逻辑继续适用），并同步更新所有产物路径。
- 关于 `--with-gauxc=no`：上游 `install_libtorch.sh` 使用固定 sha256 的 `libtorch-cxx11-abi-shared-with-deps-2.7.1+cpu.zip`（x86_64），而 `--install-all` 会启用 gauxc 并强制 `with_libtorch=__INSTALL__`，使原 `--with-libtorch=no` 失效；2026.2 官方 `Dockerfile.test_arm64-psmp` 的 CMake 配置明确关闭 `CP2K_USE_ACE/LIBTORCH/GAUXC`。为避免 aarch64 链接 x86 libtorch 失败，补充关闭 gauxc/ace，`build_cp2k.sh` 会自动据此生成对应 `-DCP2K_USE_*=OFF`。
- 注：本分支已存在的 `xz` 依赖修复（libint 在 2026.2 改为 `.tar.xz`，日志中 `Installing LIBINT ... Step libint took 217.00 seconds` 已成功）予以保留，本次未改动。

## 潜在风险
- 本次仅按上游 2026.2 官方 CMake 流程改写，未能在此环境实际执行完整 Docker 构建（CP2K 全量编译耗时长）；构建脚本 `build_cp2k.sh`、`do_regtest.py` 的接口均依据 `support/v2026.2` 源码逐一核对。
- 关闭 `ACE`/`GAUXC`（及其依赖 libtorch）会移除对应可选功能，符合原 Dockerfile `--with-libtorch=no` 的意图与上游 arm64 profile 的做法，不影响基础 cp2k 计算功能。
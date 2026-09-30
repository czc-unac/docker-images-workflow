# 修复摘要

## 修复的问题
修复 milvus 3.0.2 在 openEuler 24.03-LTS-SP4 上构建时，Conan 编译 aws-c-cal 阶段因系统 `cmake` 加载到 Conan 自带 OpenSSL 而崩溃（`cmake: symbol lookup error: /lib64/libldap.so.2: undefined symbol: EVP_md2, version OPENSSL_3.0.0`）的问题。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`:
  - 从首个 `yum install` 包列表中移除 `cmake`（不再使用 openEuler 发行版自带 cmake）。
  - 新增一步：下载官方 CMake 3.31.8 预编译包解压到 `/usr/local`（与上游 milvus `build/docker/builder/cpu/ubuntu22.04/Dockerfile` 使用的 3.31.8 一致），并显式把 `/usr/local/bin` 加入 `PATH`，保证 Conan/`install_deps.sh` 使用该 cmake。
  - 保留原有的 `install_deps.sh` 发行版分支 patch（`amzn)` → `amzn|openEuler|openeuler)`）。

## 修复逻辑
- 实际 CI 失败定位（非旧报告中的发行版检测失败）：修复分支最新构建 x86_64 构建 #4855、aarch64 构建 #4951 已通过发行版检测（日志 `[INFO] CMake 3.31 already installed`，说明 sed patch 生效），随后在 `make build-cpp` 的 3rdparty/Conan 阶段失败：
  - 失败包：`aws-c-cal/0.9.14`（依赖 `openssl/3.3.Z`）。
  - 直接错误：`cmake: symbol lookup error: /lib64/libldap.so.2: undefined symbol: EVP_md2, version OPENSSL_3.0.0` → `ConanException: Error 127 while executing cmake.configure()` → `make: *** [Makefile:305: build-3rdparty] Error 1`。
- 根因：openEuler 的发行版 `cmake`（`/usr/bin/cmake`，3.31.12）动态链接了系统 `libcurl → libldap.so.2 → libssl`。milvus 的 `scripts/3rdparty_build.sh` 会生成 Conan `pre_build` hook（`hook_fix_shared_lib_env.py`），在构建每个包时把其依赖包的 `lib` 目录前插到 `LD_LIBRARY_PATH`；当包依赖 `openssl/3.3.Z`（`openssl/*:shared=True`）时，`LD_LIBRARY_PATH` 中的 Conan OpenSSL 会覆盖系统 OpenSSL，导致系统 `libldap` 解析不到 `EVP_md2@OPENSSL_3.0.0`，从而使系统 `cmake` 进程直接崩溃。此前较早的包（如 benchmark）不依赖 OpenSSL，故未触发；到 `aws-c-cal`（第 60/90 个包）才暴露。
- 修复方式：改用官方 CMake 预编译二进制。已验证 `https://cmake.org/files/v3.31/cmake-3.31.8-linux-x86_64.tar.gz` 中的 `bin/cmake`：
  - `objdump -p bin/cmake | grep NEEDED` 仅有 `libdl/librt/libpthread/libm/libc/ld-linux`，**不链接 libcurl/libldap/libssl**（也不需要 libstdc++）；
  - 所需最高 `GLIBC_2.17`，openEuler 24.03 的 glibc 可满足；
  - aarch64 同名包 `cmake-3.31.8-linux-aarch64.tar.gz` 同样存在（HTTP 200）。
  因此 Conan 构建时无论 `LD_LIBRARY_PATH` 是否被注入 Conan OpenSSL，cmake 都不会再加载 `libldap`，从根上消除该符号冲突。同时 `install_deps.sh` 的 `install_cmake_linux()` 会检测到 `/usr/local/bin/cmake` 为 3.31（≥3.26）并跳过安装，保持行为一致。
- 上游一致性：milvus v3.0.2 官方 builder（ubuntu22.04）即使用 cmake 3.31.8（`Wget cmake.org/files/v3.31/cmake-3.31.8`），本次改为官方 cmake 与该路径一致。
- 正则/上游文件验证（沿用并复核）：已从 `https://raw.githubusercontent.com/milvus-io/milvus/v3.0.2/scripts/install_deps.sh` 获取真实源码，确认 `main()` 的 `case "$distro"` 中确有 `amzn)` 分支；`sed -i -E 's/^([[:space:]]*)amzn\)/\1amzn|openEuler|openeuler)/'` 经测试可精确匹配且仅替换 1 处，结果 `amzn|openEuler|openeuler)` 语法正确；实际 CI 日志也确认该 patch 已使发行版检测通过。
- 运行时阶段 URL 复核：旧修复把 minio 下载地址改为 `https://dl.min.io/aistor/minio/release/linux-$TARGETARCH/minio`，实为必要修正（原 `dl.min.io/server/...` 现返回 HTTP 410 Gone，`aistor` 路径返回 206 有效），保留不动。

## 潜在风险
- 仍依赖 openEuler 其余构建工具链与上游基准（Ubuntu/Rocky/Amazon）的差异；若后续 3rdparty 编译再出现与发行版相关的兼容问题，属于本次修复之后才会暴露的下一层问题，不在本 CI 失败根因范围内。
- 官方 CMake 版本为 3.31.8，略低于 openEuler 自带的 3.31.12；milvus v3.0.2 官方 ubuntu builder 同样使用 3.31.8，兼容性风险很低。
- 该修复只解决系统 `cmake` 加载 Conan OpenSSL 的冲突；若后续某个依赖 OpenSSL 的包在构建过程中调用其它链接 `libldap` 的系统工具，理论上仍可能触发同类冲突，但当前日志中仅 `cmake` 命中该错误。
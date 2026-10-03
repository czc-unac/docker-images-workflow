# 修复摘要

## 修复的问题
milvus 3.0.2 镜像在 builder 阶段执行 `make build-cpp`（conan 编译第三方依赖 `aws-c-cal`）时，使用的 openEuler 系统 `cmake` 因链接 `libcurl -> libldap` 而报错 `cmake: symbol lookup error: /lib64/libldap.so.2: undefined symbol: EVP_md2, version OPENSSL_3.0.0`，导致两架构构建失败。改用官方静态 CMake 3.27.5（不依赖 libldap）修复。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`:
  - 从第一条 `yum install` 中移除系统包 `cmake`（openEuler 3.31.12 动态链接 `libcurl/libldap`）。
  - 新增下载并解压官方静态 CMake 3.27.5 到 `/usr/local`（`https://cmake.org/files/v3.27/cmake-3.27.5-linux-$(uname -m).tar.gz`），使其在 PATH 中优先于系统 cmake。

## 修复逻辑
- 提供的分析报告无日志、置信度为低。为定位真实根因，实际拉取了 CI 失败 job 的完整构建日志：
  - 原 PR HEAD 构建：`x86-64 #4965` 失败于 minio 下载 `curl: (22) ... error: 410`（`dl.min.io` 对社区二进制已全面返回 410，MinIO 已归档）。
  - 当前 fix 分支构建：`x86-64 #4991` / `aarch64 #5087` 均失败于 `cmake: symbol lookup error: /lib64/libldap.so.2: undefined symbol: EVP_md2, version OPENSSL_3.0.0`（发生在 conan 构建 `aws-c-cal/0.9.14` 的 `cmake.configure()`）。
- 根因：openEuler 基础镜像自带的 `cmake-3.31.12` 动态依赖 `libcurl.so.4 -> libssl.so.3/libcrypto.so.3` 与 `libldap.so.2`。conan 的 `pre_build` hook（`hook_fix_shared_lib_env.py`）将依赖库目录（含 conan 自建的 `openssl/3.3.2`）前置到 `LD_LIBRARY_PATH`，导致系统 `libldap.so.2` 解析到 conan 的 `libcrypto`，而该 openssl 不导出 `EVP_md2`，从而符号查找失败。
- 修复依据：Milvus 官方 builder（`build/docker/builder/cpu/rockylinux9/Dockerfile`）正是将官方静态 CMake 3.27.5 安装到 `/usr/local` 且不使用发行版 cmake。经实测（`objdump -p`），官方 `cmake-3.27.5-linux-x86_64/bin/cmake` 仅依赖 `libdl/librt/libpthread/libm/libc`，不含 `libcurl/libldap`，因此彻底规避该符号冲突；同时 `cmake.org` 对 `x86_64` 与 `aarch64` 均提供 3.27.5 包（已确认 HTTP 200）。
- 移除系统 `cmake` 可保证构建过程中唯一可用的 cmake 就是 `/usr/local/bin/cmake`（PATH 中 `/usr/local/bin` 先于 `/usr/bin`），避免 conan 的 `shutil.which("cmake")` 再次选中系统版本；上游 `scripts/install_deps.sh` 的 `install_cmake_linux` 会检测到 `/usr/local/bin/cmake` (3.27.5 >= 3.26) 而跳过重复下载。
- 上游验证：已从 `milvus-io/milvus` v3.0.2 获取 `scripts/install_deps.sh`，确认既有正则 patch 均能匹配（`sudo dnf install -y epel-release dnf-plugins-core`、`sudo dnf config-manager --set-enabled crb`、` ccache lcov libtool` 以及 `/etc/os-release` 的 `ID=` 行）。

## 潜在风险
- MinIO 已归档全部社区二进制，`dl.min.io` 对任意路径返回 410；现有（上一轮修复引入的）`mirrors.huaweicloud.com` 路径实际返回的是镜像站 SPA 的 HTML 页面（HTTP 200），`curl -fSL` 不会报错，因此不会导致 CI 构建失败，但产物中的 `/usr/bin/minio` 并非有效二进制。该问题不影响本次 `check_build` 通过，且不在本次 CI 失败链路上，故未改动；若后续需要可运行的 minio，建议改为从 `minio/minio` 容器镜像 `COPY --from` 二进制。
- `check_package_license` 在 CI 结果中为 WARNING（“缺少项目级Copyright声明文件”），非本次构建失败原因，且属仓库级既有问题，未做处理。
- 该修复仅解决 `aws-c-cal` 报出的 `cmake` 符号冲突；后续其他依赖 conan openssl 且经 PATH 调用系统 cmake 的包同样会受益于官方静态 cmake，理论上不会再触发同一错误。
# 修复摘要

## 修复的问题
为 meryl 1.4.2 镜像补齐编译内嵌 htslib 所需的开发库依赖，解决 `zlib.h: No such file or directory` 构建失败。

## 修改的文件
- `HPC/meryl/1.4.2/24.03-lts-sp4/Dockerfile`: 在首个 `yum install` 步骤中补充 `zlib-devel bzip2-devel xz-devel libcurl-devel openssl-devel`，均位于 `make` 之前生效。

## 修复逻辑
分析报告的根因成立：该 Dockerfile 仅安装 `git gcc gcc-c++ make which`，缺少 meryl 内嵌 htslib 编译所需的开发库，导致 `#include <zlib.h>` 失败。

已从上游实际源文件核实依赖集（meryl `v1.4.2` 的 utility 子模块 commit `f800fc4dada365a701122b5ac1ecb946eb402cb4`）：
- `meryl-utility/src/htslib/config.h` 中定义了 `HAVE_LIBBZ2`、`HAVE_LIBLZMA`、`HAVE_LZMA_H`、`HAVE_HMAC`、`HAVE_LIBCURL`；
- `meryl-utility/src/main.mk` 中显式执行 `pkg-config --cflags-only-I openssl libcurl liblzma`，且 `LDLIBS += ... -lz -lbz2`，并编译 `htslib/hts/hfile_libcurl.c` 等源文件。

据此，除直接报错的 zlib（`zlib.h`/`-lz`）外，还硬性需要：
- `bzip2-devel`：提供 `bzlib.h`/`-lbz2`（`HAVE_LIBBZ2`，`LDLIBS` 无条件链接）；
- `xz-devel`：提供 `lzma.h`/`liblzma`（`HAVE_LIBLZMA`/`HAVE_LZMA_H`，日志中 pkg-config `liblzma` not found）；
- `libcurl-devel`：提供 curl 头文件与 `libcurl`（`HAVE_LIBCURL`，日志中 pkg-config `libcurl` not found）；
- `openssl-devel`：提供 `libcrypto`/`libssl`（`HAVE_HMAC`，日志中 pkg-config `openssl` not found）。

以上包名均为 openEuler 仓库标准 `-devel` 包名，且在本仓库其他 openEuler 镜像 Dockerfile 中被广泛使用（zlib-devel/bzip2-devel/xz-devel/libcurl-devel/openssl-devel 均有先例）。修改一次性补齐，避免 CI 反复暴露后续缺库错误。`pkg-config` 在失败日志中已能输出 not found 信息，说明基础镜像已自带，无需额外安装。

## 潜在风险
- 新增依赖仅在构建阶段使用，未加入运行时清理逻辑去除构建工具，符合原有 Dockerfile 风格，不影响运行镜像功能。
- aarch64 分支的 `printf '' > align-*.C` 处理逻辑未改动；该分支能否通过需依赖 arm64 job 实际验证。
- 缺失的 `git clone` tag 提示（`refs/tags/v1.4.2 ... is not a commit`）为非致命提示，未处理。
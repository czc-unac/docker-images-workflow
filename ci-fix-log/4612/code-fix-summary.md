# 修复摘要

## 修复的问题
meryl 1.4.2 Dockerfile 未安装 htslib 编译所依赖的开发包，导致 `utility/src/htslib/hts/bgzf.c:38` 无法找到 `zlib.h` 而编译失败。

## 修改的文件
- `HPC/meryl/1.4.2/24.03-lts-sp4/Dockerfile`: 在 `yum install` 列表中补充 htslib 所需的开发包 `zlib-devel bzip2-devel xz-devel libcurl-devel openssl-devel`。

## 修复逻辑
- 失败根因：meryl 1.4.1 没有该 Dockerfile，1.4.2 随源码树内嵌了 htslib。构建执行 `make`，编译单元 `htslib/hts/bgzf.c` 直接 `#include <zlib.h>`，但镜像中只有 `git gcc gcc-c++ make which`，缺少 zlib 开发头文件，编译在第一步 htslib 单元即终止（exit code 2）。这与分析报告方向 1 完全一致。
- 补包依据（已从上游验证实际依赖，避免二次失败）：
  - 上游 meryl v1.4.2 的子模块 meryl-utility 固定在 commit `f800fc4`。其 `src/Makefile`（main.mk）中显式依赖：
    `SYS_INCDIRS/LDFLAGS/LDLIBS += $(shell pkg-config ... openssl libcurl liblzma) -lz -lbz2`
    即需要 `openssl`、`libcurl`、`liblzma` 的 pkg-config `.pc` 文件，以及 `zlib`、`bzip2` 链接库。
  - 其 `src/htslib/config.h` 硬编码启用：`HAVE_LIBBZ2`、`HAVE_LIBLZMA`、`HAVE_LZMA_H`、`HAVE_HMAC`、`HAVE_LIBCURL`，对应源码会 `#include <bzlib.h>`、`<lzma.h>`、`<curl/curl.h>`、`<openssl/hmac.h>`，故这些 `-devel` 包缺一不可。
  - 构建日志中已出现 `Package liblzma was not found`、`Package 'libcurl' ... not found`、`Package 'openssl' ... not found` 的 pkg-config 告警，佐证上述 `.pc` 依赖缺失。
  - 已从上游 `marbl/meryl-utility` 对应 commit 获取 `src/Makefile`、`src/htslib/config.h`、`src/htslib/config_vars.h` 核对，依赖集合确认无误（本修复不涉及正则 patch 外部源文件）。
- 包名与 openEuler 24.03-LTS-SP4 仓库及其它镜像（如 `HPC/lumpy`、`HPC/hdwgcna`、`Database/tdengine`）保持一致：`zlib-devel`、`bzip2-devel`、`xz-devel`（提供 liblzma）、`libcurl-devel`、`openssl-devel`。

## 潜在风险
- 仅在构建阶段安装开发包，未引入运行时依赖变更；镜像体积略有增加。
- 若 aarch64 分支执行相同的 make 流程，同样受益于此修复（原错误与架构无关）。
# 修复摘要

## 修复的问题
为 meryl 1.4.2 新增 Dockerfile 补齐编译所需开发依赖包，解决 htslib 编译 `bgzf.c` 时 `zlib.h` 缺失导致的 build-error。

## 修改的文件
- `HPC/meryl/1.4.2/24.03-lts-sp4/Dockerfile`: 在 `yum install` 步骤中新增 `zlib-devel xz-devel libcurl-devel openssl-devel pkgconf-pkg-config` 五个开发/工具包。

## 修复逻辑
CI 失败日志显示：pkg-config 找不到 `liblzma`、`libcurl`、`openssl`，随后 htslib 的 `utility/src/htslib/hts/bgzf.c:38` `#include <zlib.h>` 直接 fatal error，make 退出码 2。

原 Dockerfile 只安装了 `git gcc gcc-c++ make which`，缺少开发库头文件与 `.pc` 文件。meryl 1.4.2 相对 1.4.1 引入了对 htslib/压缩库的构建需求（1.4.1 相同依赖清单可通过，说明 1.4.2 构建要求增加，对应分析报告方向 2）。本次按分析报告方向 1 补齐：

- `zlib-devel` → 提供 `zlib.h` / `zlib.pc`，直接修复 fatal error；
- `xz-devel` → 提供 `liblzma` / `liblzma.pc`；
- `libcurl-devel` → 提供 `libcurl.pc`；
- `openssl-devel` → 提供 `openssl.pc`；
- `pkgconf-pkg-config` → 提供 `pkg-config`，确保上述 `.pc` 文件可被检索。

同时将原 `which &&` 行尾续行改为 `which \` 并另起一行列出新增包，保证 shell 续行语法正确、构建命令仍为单条 RUN 安装。

## 验证情况
- 不涉及正则 patch 外部源文件，无需从上游拉取文件校验正则。
- 已确认所补包名在 openEuler 24.03-LTS-SP4 仓库可用（同仓 SP4 Dockerfile 中均有使用先例）：
  - `pkgconf-pkg-config`、`libcurl-devel`：`Cloud/libvirt/12.7.0/24.03-lts-sp4/Dockerfile`
  - `xz-devel`：`Storage/3fs/22fca04/24.03-lts-sp4/Dockerfile`
  - `zlib-devel`、`openssl-devel`：仓库内大量 SP4 Dockerfile 使用。
- 修复对 x86_64 与 aarch64 均适用（依赖包与架构无关，aarch64 专属补丁逻辑未改动）。

## 潜在风险
无。仅新增构建期开发依赖，不改变运行时行为；`yum clean all` 保证不增大最终镜像层。
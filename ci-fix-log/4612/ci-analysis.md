# CI 失败分析报告

## 基本信息
- PR: #4612 — 【自动升级】meryl容器镜像升级至1.4.2版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式10（缺少构建依赖 CMake/configure 找不到系统库）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#10 7.552 cc -o /meryl/build/obj/lib/libmeryl.a/utility/src/htslib/hts/bgzf.o ... utility/src/htslib/hts/bgzf.c
#10 7.637 utility/src/htslib/hts/bgzf.c:38:10: fatal error: zlib.h: No such file or directory
#10 7.637    38 | #include <zlib.h>
#10 7.637       |          ^~~~~~~~
#10 7.637 compilation terminated.
#10 7.638 make: *** [Makefile:417: /meryl/build/obj/lib/libmeryl.a/utility/src/htslib/hts/bgzf.o] Error 1
#10 7.638 make: *** Waiting for unfinished jobs....
#10 ERROR: process "... make -j 4 && cp /meryl/build/bin/meryl /usr/bin/" did not complete successfully: exit code: 2
```
同一构建步骤早期还出现依赖探测缺失告警：
```
#10 0.134 Package liblzma was not found in the pkg-config search path.
#10 0.134 Package 'libcurl', required by 'virtual:world', not found
#10 0.134 Package 'openssl', required by 'virtual:world', not found
```

### 根因定位
- 失败位置: `HPC/meryl/1.4.2/24.03-lts-sp4/Dockerfile:10`（RUN make 步骤），具体编译单元 `utility/src/htslib/hts/bgzf.c:38`
- 失败原因: Dockerfile 的 `yum install` 只安装了 `git gcc gcc-c++ make which`，未安装 htslib 编译所需的 `zlib-devel`（及其它 pkg-config 依赖库的开发包），导致 `#include <zlib.h>` 找不到头文件而编译终止。

### 与 PR 变更的关联
本次 PR 新增了 `HPC/meryl/1.4.2/24.03-lts-sp4/Dockerfile`（1.4.1 时不存在该 Dockerfile，属全新文件）。新增 Dockerfile 的第 5-6 行依赖安装列表缺少 zlib 等开发包，而 meryl 1.4.2 源码树内嵌的 htslib 在 `bgzf.c` 中直接 `#include <zlib.h>`，因此该 PR 的改动直接引入了此失败。构建其余部分（git clone、aarch64 空文件处理、g++ 编译数百个文件）均正常，仅在编译 htslib 时中断。

## 修复方向

### 方向 1（置信度: 高）
在新增 Dockerfile 的 `yum install` 步骤中补充 htslib 编译所需的开发包，至少包括 `zlib-devel`；根据日志中 pkg-config 探测失败项，一并考虑 `xz-devel`（liblzma）、`libcurl-devel`、`openssl-devel`，以确保后续链接/配置阶段不再因缺库失败。

### 方向 2（可选）
若上游 htslib 仅需 zlib，可只补 `zlib-devel` 最小化改动后重新触发构建；若仍报 liblzma/libcurl/openssl 相关错误，再按方向 1 补齐其余 `-devel` 包。

## 需要进一步确认的点
- meryl 1.4.2 内嵌 htslib 的实际链接依赖集合（是否强制依赖 libcurl/openssl/liblzma），以确定需要补齐的 `-devel` 包完整列表，避免二次失败。
- `meta.yml` 中新增的 `1.4.2-oe2403sp4` 条目是否需要 `arch` 约束——本日志为 x86_64 上 `uname -m` 判断，aarch64 分支另行处理，但当前错误与架构无关，两架构均会缺 zlib.h。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
不适用。本失败源于 Dockerfile 依赖安装列表，不涉及对第三方源文件的正则 patch。

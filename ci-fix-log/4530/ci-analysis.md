# CI 失败分析报告

## 基本信息
- PR: #4530 — 【自动升级】meryl容器镜像升级至1.4.2版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式10
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#10 13.26 utility/src/htslib/hts/bgzf.c:38:10: fatal error: zlib.h: No such file or directory
#10 13.26    38 | #include <zlib.h>
#10 13.26       |          ^~~~~~~~
#10 13.26 compilation terminated.
#10 13.26 make: *** [Makefile:417: /meryl/build/obj/lib/libmeryl.a/utility/src/htslib/hts/bgzf.o] Error 1
#10 ERROR: process "/bin/sh -c if [ \"$(uname -m)\" = \"aarch64\" ]; then ... fi && make -j 4 && cp /meryl/build/bin/meryl /usr/bin/" did not complete successfully: exit code: 2
ERROR: failed to solve: ... exit code: 2
```

日志结尾为 `Build step 'Execute shell' marked build as failure` / `Finished: FAILURE`，确为真实构建失败（非 trigger 层成功、下游不明的情况），可正常定位根因。

### 根因定位
- 失败位置: `HPC/meryl/1.4.2/24.03-lts-sp4/Dockerfile:10`（`make -j 4` 步骤），根本缺失项在 `Dockerfile:5` 的 `yum install` 依赖列表
- 失败原因: Dockerfile 仅安装了 `git gcc gcc-c++ make which`，未安装 zlib 开发头文件（`zlib-devel`），导致 meryl 编译其内嵌 htslib 时 `#include <zlib.h>` 找不到头文件而编译终止。

### 与 PR 变更的关联
本 PR 新增 `HPC/meryl/1.4.2/24.03-lts-sp4/Dockerfile`（全新文件），其中 `RUN yum install -y git gcc gcc-c++ make which` 的依赖列表不完整，直接触发本次失败。属本次 PR 引入的问题，与历史版本无关。

补充证据：在同一步骤早期，pkg-config 已报告多个库缺失，进一步印证是构建依赖缺失而非上游源码问题：
```
#10 0.167 Package 'liblzma', required by 'virtual:world', not found
#10 0.167 Package 'libcurl', required by 'virtual:world', not found
#10 0.167 Package 'openssl', required by 'virtual:world', not found
```
注：`#8` 中 `git clone` 的 `warning: refs/tags/v1.4.2 ... is not a commit!` 为非致命提示，克隆已成功切换到 commit `dbacc3d...`，不是本次失败根因。

## 修复方向

### 方向 1（置信度: 高）
在 Dockerfile 首个 `yum install` 步骤中补齐编译 meryl/htslib 所需的 `-devel` 依赖，至少补入提供 `zlib.h` 的 `zlib-devel`；结合 pkg-config 缺失提示，建议同时补齐 `xz-devel`（liblzma）、`libcurl-devel`、`openssl-devel`，避免后续逐步暴露新的缺库错误。

### 方向 2（可选）
若不确定完整依赖集，可参照上游 meryl/meryl-utility 项目的构建依赖文档或同仓库其他 HPC 镜像的依赖清单，确认 htslib 所需的全部开发库后再一次性补齐，减少多次 CI 往返。

## 需要进一步确认的点
- `zlib-devel` 在 `openeuler/openeuler:24.03-lts-sp4` 仓库中的确切包名是否为 `zlib-devel`（openEuler 通常为此名，但需确认）。
- meryl 1.4.2 / htslib 除 zlib 外是否还硬性依赖 `liblzma`、`libcurl`、`openssl`、`bzip2`；日志已出现这三者的 pkg-config not found，若上游 Makefile 对缺失库采用条件编译则可能非致命，建议 code-fixer 一并核实，避免修复不彻底。
- 本失败发生在 x86_64（日志显示 `as 'amd64'`）；新增 Dockerfile 同时面向 amd64/arm64，aarch64 分支的 `printf '' > align-*.C` 处理逻辑是否仍适用于 1.4.2 的源码结构，需在依赖修复后由 arm64 job 验证。

## 修复验证要求
本次修复为在 Dockerfile 中补充系统 `-devel` 包，不涉及对第三方源文件的正则 patch，无需额外上游文件匹配验证。code-fixer 修改后应确认 `zlib-devel`（及补充的其他 `-devel` 包）位于 yum 步骤且在 `make` 之前生效。

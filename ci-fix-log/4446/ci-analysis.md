# CI 失败分析报告

## 基本信息
- PR: #4446 — 【自动升级】meryl容器镜像升级至1.4.2版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式10（缺少构建依赖 / 编译找不到系统库）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#10 0.168 Package liblzma was not found in the pkg-config search path.
#10 0.169 Package 'libcurl', required by 'virtual:world', not found
#10 0.169 Package 'openssl', required by 'virtual:world', not found
#10 13.29 cc -o /meryl/build/obj/lib/libmeryl.a/utility/src/htslib/hts/bgzf.o ... utility/src/htslib/hts/bgzf.c
#10 13.32 utility/src/htslib/hts/bgzf.c:38:10: fatal error: zlib.h: No such file or directory
#10 13.32    38 | #include <zlib.h>
#10 13.32       |          ^~~~~~~~
#10 13.32 compilation terminated.
#10 13.32 make: *** [Makefile:417: /meryl/build/obj/lib/libmeryl.a/utility/src/htslib/hts/bgzf.o] Error 1
#10 ERROR: process "/bin/sh -c ... make -j 4 ..." did not complete successfully: exit code: 2
```

### 根因定位
- 失败位置: 编译阶段 `utility/src/htslib/hts/bgzf.c:38`（`#include <zlib.h>`），源头在新增 Dockerfile 第 5-6 行的 `yum install` 步骤
- 失败原因: 新增的 `HPC/meryl/1.4.2/24.03-lts-sp4/Dockerfile` 只安装了 `git gcc gcc-c++ make which`，缺少 meryl 构建所需的开发库头文件/`pkg-config` 文件。构建早期即出现 `liblzma`、`libcurl`、`openssl` 三个 pkg-config 包找不到的提示，随后 htslib 编译 `bgzf.c` 时因找不到 `zlib.h` 直接 fatal error，make 返回 exit code 2，整层 RUN 失败。

### 与 PR 变更的关联
本 PR 为新增 1.4.2 版本的 Dockerfile（`HPC/meryl/1.4.2/24.03-lts-sp4/Dockerfile`，全新文件 18 行）。构建依赖清单在新文件中被漏写，导致依赖头文件缺失。失败直接由本次 PR 引入的新 Dockerfile 引起，与上游 meryl 1.4.2 源码本身无关（clone 与 submodule 均成功）。

## 修复方向

### 方向 1（置信度: 高）
在 `HPC/meryl/1.4.2/24.03-lts-sp4/Dockerfile` 的 `yum install` 步骤中补齐构建依赖的开发包，重点覆盖日志中 pkg-config 报缺失及编译报缺失的库：
- zlib 开发包（对应 `zlib.h` / `zlib.pc`）
- lzma/xz 开发包（对应 `liblzma`）
- libcurl 开发包（对应 `libcurl`）
- openssl 开发包（对应 `openssl`）
- 建议同时补齐 `pkgconf-pkg-config`，确保 pkg-config 能检索到上述 `.pc` 文件

应参照同目录/同项目 1.4.1 版本 Dockerfile 的依赖清单，保持依赖集合一致。

### 方向 2（可选）
若上游 meryl 1.4.2 相比 1.4.1 新增了对 htslib/压缩库的构建要求（新增 `bgzf.c` 等），则所需依赖可能多于 1.4.1 的清单，需按 meryl 1.4.2 的实际构建要求确认完整依赖集合，而非仅照搬旧版本。

## 需要进一步确认的点
1. `HPC/meryl/1.4.1/24.03-lts-sp4/Dockerfile` 中的 `yum install` 依赖清单具体包含哪些包，用于对齐 1.4.2 的依赖。
2. meryl 1.4.2 上游（marbl/meryl，tag v1.4.2）构建 htslib 时对 zlib/lzma/curl/openssl 的确切需求，以确认除 zlib 外是否还需 lzma/curl/openssl 开发包（日志中 pkg-config 已提示缺失，倾向需要）。
3. `pkg-config`/`pkgconf-pkg-config` 是否已在基础镜像中，若缺失需一并安装，否则即使装了 `-devel` 包也可能检索不到 `.pc`。
4. 该失败在 x86_64 与 aarch64 上是否均复现（日志为 amd64 构建环境，`For 'Linux' ... as 'amd64'`），确认修复对两架构均适用。

## 修复验证要求
本修复不涉及正则 patch 外部源文件，无需从上游拉取文件校验正则。建议 code-fixer 提交前确认所补依赖包名在 openEuler 24.03-LTS-SP4 仓库中可用，并尽量与 meryl 1.4.1 版本现有 Dockerfile 的依赖写法保持一致。

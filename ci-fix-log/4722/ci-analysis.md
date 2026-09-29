# CI 失败分析报告

## 基本信息
- PR: #4722 — 【自动升级】bolt容器镜像升级至2d01261版本
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式10（缺少构建依赖 / CMake 找不到系统库）
- 新模式标题: (匹配已有模式，不填)
- 新模式症状关键词: (匹配已有模式，不填)

## 根因分析

### 直接错误
```
#12 1751.9 /root/.conan2/p/b/cmake48959872aef33/b/src/Utilities/cm3p/zlib.h:8:12: fatal error: zlib.h: No such file or directory
#12 1751.9     8 | #  include <zlib.h> // IWYU pragma: export
#12 1751.9       |            ^~~~~~~~
#12 1751.9 compilation terminated.
#12 1751.9 gmake[4]: *** [Utilities/cmcurl/lib/CMakeFiles/cmcurl.dir/build.make:303: Utilities/cmcurl/lib/CMakeFiles/cmcurl.dir/content_encoding.c.o] Error 1
#12 1751.9 gmake[3]: *** [CMakeFiles/Makefile2:834: Utilities/cmcurl/lib/CMakeFiles/cmcurl.dir/all] Error 2
#12 1751.9 gmake[2]: *** [Makefile:156: all] Error 2
#12 1751.9 cmake/3.31.10: ERROR: Package 'b92cbfaeb3f7ac7a33a54961f77681b33fdffbd5' build failed
#12 1751.9 ERROR: cmake/3.31.10: Error in build() method, line 148
#12 1751.9 	ConanException: Error 2 while executing
#12 1751.9 make[1]: *** [Makefile:257: conan_build] Error 1
#12 1751.9 make: *** [Makefile:315: release] Error 2
#12 ERROR: process "/bin/sh -c bash scripts/install-bolt-deps.sh && conan profile detect && make release && make export_release" did not complete successfully: exit code: 2
```

### 根因定位
- 失败位置: `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile:21`（`RUN bash scripts/install-bolt-deps.sh && conan profile detect && make release && make export_release`）
- 失败原因: conan 在构建依赖 `cmake/3.31.10` 时，编译 cmake 内置的 `cmcurl` 组件（`Utilities/cmcurl/lib/content_encoding.c`），其转发头 `Utilities/cm3p/zlib.h` 需要包含系统 `<zlib.h>`，但 openEuler 基础镜像中未安装 `zlib-devel`，编译器报 `fatal error: zlib.h: No such file or directory`，进而导致 `conan_build` → `make release` 失败（exit code 2）。

### 与 PR 变更的关联
- 本次 PR 新增 Dockerfile `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile`，其 `dnf install` 依赖清单为 `git gcc gcc-c++ make cmake ninja-build patch libstdc++-static glibc-static curl python3-pip`，**不包含 `zlib-devel`**。构建流程随后执行 `make release` 触发 conan 构建 cmake，直接暴露该缺失依赖。
- 因此该失败由本 PR 新增的构建步骤直接触发，属于 PR 相关问题（新增镜像首次构建）。
- 日志中其余信息均为非致命噪声，不可作为根因：
  - `fatal: not a git repository (or any of the parent directories): .git` —— conan recipe 获取源码版本信息时的常规提示，构建仍在继续。
  - 大量 `-- Performing Test ... - Failed` —— CMake 特性探测的正常结果。
  - 时间戳 `1751.9` 反复出现，是 conan 缓存/缓冲导致的输出复用，不代表失败时刻。

## 修复方向

### 方向 1（置信度: 高）
在 Dockerfile 第一个 `dnf install` 步骤中补充 zlib 开发包（openEuler 中包名为 `zlib-devel`），使 cmake 内置 cmcurl 能正确包含 `<zlib.h>`，从而让 conan 成功构建 cmake 并继续 bolt 的 `make release`。
（若 `scripts/install-bolt-deps.sh` 内部本应负责安装系统依赖，则应确认该脚本是否遗漏 zlib 开发包，并在 Dockerfile 中显式补齐以保证确定性。）

### 方向 2（置信度: 中）
该 cmake 由 conan 从源码构建（触发大量 bundled 库编译，耗时长且脆弱）。可考虑在 Dockerfile 中通过 `conan profile` / conan 配置直接使用系统已安装的 `cmake`（镜像已 `dnf install cmake`），避免从源码构建 cmake，从根本上绕开 zlib 头文件缺失问题。需确认 bolt 的 conan 依赖是否强制要求 `cmake/3.31.10` 特定版本。

## 需要进一步确认的点
- 查看上游 `bytedance/bolt` 在 `2d01261` 版本的 `scripts/install-bolt-deps.sh` 与 `conanfile`，确认：
  1. 该脚本是否应安装 `zlib-devel`（若应安装却未安装，则问题在脚本，Dockerfile 只需补齐）；
  2. 是谁/哪个依赖触发了 `cmake/3.31.10` 的 conan 源码构建，以及是否可用系统 cmake 替代。
- 确认 bolt 项目构建除 zlib 外是否还依赖其他 `-devel` 包（如 bzip2-devel、libarchive-devel、openssl-devel 等），避免逐个暴露后反复失败。
- 确认基础镜像 `openeuler/openeuler:24.03-lts-sp4` 默认仓库中 `zlib-devel` 包名与可用性。

## 修复验证要求
本失败不涉及“修改正则匹配第三方/上游源文件”，无需执行对应的上游拉取校验。code-fixer 完成后，应至少重新触发一次该 Dockerfile 的完整构建（含 x86-64 与 aarch64），确认 conan 构建 cmake 阶段不再出现 `zlib.h: No such file or directory`。

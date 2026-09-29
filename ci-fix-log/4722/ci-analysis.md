# CI 失败分析报告

## 基本信息
- PR: #4722 — 【自动升级】bolt容器镜像升级至2d01261版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: Boost缺quadmath头文件
- 新模式症状关键词: quadmath.h, No such file or directory, boost/charconv, libquadmath-devel, fatal error

## 根因分析

### 直接错误
```
#12 543.8     - GCC libquadmath and __float128 support : no [2]
#12 686.8 In file included from libs/charconv/build/../src/from_chars_float_impl.hpp:9,
#12 686.8                  from libs/charconv/build/../src/from_chars.cpp:14:
#12 686.8 ./boost/charconv/detail/config.hpp:32:12: fatal error: quadmath.h: No such file or directory
#12 686.8    32 | #  include <quadmath.h>
#12 686.8       |            ^~~~~~~~~~~~
#12 686.8 compilation terminated.
#12 696.3 boost/1.85.0: ERROR: 
#12 696.3 Package '3d05c3311f889e9a4efe75ff8d85497ffa81de4a' build failed
#12 696.3 ERROR: boost/1.85.0: Error in build() method, line 1167
#12 696.3 	ConanException: Error 1 while executing
#12 696.5 make[1]: *** [Makefile:257: conan_build] Error 1
#12 696.5 make: *** [Makefile:315: release] Error 2
#12 ERROR: process "/bin/sh -c bash scripts/install-bolt-deps.sh && ... make release ..." did not complete successfully: exit code: 2
Dockerfile:21
```

### 根因定位
- 失败位置: `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile:21`（`RUN bash scripts/install-bolt-deps.sh && conan profile detect && make release && make export_release`）
- 失败原因: conan 在编译 `boost/1.85.0` 的 `charconv` 组件时，`boost/charconv/detail/config.hpp:32` 无条件 `#include <quadmath.h>`，但 `openEuler 24.03-LTS-SP4` 基础镜像内未安装提供该头文件的 `libquadmath-devel` 包，导致编译立即 `fatal error: quadmath.h: No such file or directory`，进而 boost 构建失败、`make release` 退出码 2。

### 与 PR 变更的关联
- 本 PR 为自动升级，新增 `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile`（新文件），并同步更新 `README.md`、`doc/image-info.yml`、`meta.yml`。
- 失败发生在新增 Dockerfile 的构建步骤中，属于 PR 直接引入的新构建路径。Dockerfile 的 `dnf install` 列表为 `git gcc gcc-c++ make cmake ninja-build patch libstdc++-static glibc-static curl python3-pip`，**缺少 `libquadmath-devel`**，与本次报错直接对应。
- 日志中该失败发生在 x86_64（`architecture=x86 address-model=64`）架构，说明并非架构不匹配问题，而是通用依赖缺失。

## 修复方向

### 方向 1（置信度: 高）
在新增 Dockerfile 的 `dnf install` 步骤中补充提供 `quadmath.h` 的开发包（openEuler 中通常为 `libquadmath-devel`），使 boost/charconv 能正常编译。若 `scripts/install-bolt-deps.sh` 已负责安装依赖，则应确认该脚本未覆盖/未包含此包。

### 方向 2（可选，置信度: 中）
若上游 bolt 的 conan 配置确实不需要 boost/charconv（日志中 b2 检查已报告 `GCC libquadmath and __float128 support : no`），可在 conan 选项中关闭 charconv 组件（如 `without_charconv`）。但该方式会改变产物功能，风险高于方向 1，需确认 bolt 功能不受影响。

## 需要进一步确认的点
- 确认 openEuler 24.03-LTS-SP4 中提供 `/usr/include/quadmath.h`（或 gcc include 目录下）的准确包名是否为 `libquadmath-devel`，以及其在 aarch64 架构上是否同样可用（若 aarch64 无该包，需评估 charconv 条件编译差异）。
- 确认 `scripts/install-bolt-deps.sh` 是否会安装/覆盖系统依赖，避免修复被脚本行为抵消。
- 本次提供的日志仅为 x86_64 构建 job，若 aarch64 job 报错不同，需补充其日志单独确认。

## 修复验证要求
- 置信度为"高"，但仍要求 code-fixer 在提交前本地/CI 验证：在 `openEuler 24.03-LTS-SP4` 基础镜像中安装所补充的包后，`quadmath.h` 可被 gcc 找到（例如 `gcc -E -x c - <<< '#include <quadmath.h>'`），再执行 bolt 的 `make release` 确认 boost/charconv 编译通过。
- 若采用方向 2，必须从 bolt `2d01261` 上游仓库确认 conan 选项名及关闭 charconv 对 bolt 功能/产物的影响。

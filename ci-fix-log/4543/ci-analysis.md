# CI 失败分析报告

## 基本信息
- PR: #4543 — 【自动升级】bolt容器镜像升级至2d01261版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 模式10（缺少构建依赖，编译期头文件缺失变体）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```text
#12 364.6 In file included from ./boost/charconv/detail/emulated128.hpp:11,
#12 364.6                  from ./boost/charconv/detail/apply_sign.hpp:9,
#12 364.6                  from libs/charconv/build/../src/to_chars_float_impl.hpp:10,
#12 364.6                  from libs/charconv/build/../src/to_chars.cpp:7:
#12 364.6 ./boost/charconv/detail/config.hpp:32:12: fatal error: quadmath.h: No such file or directory
#12 364.6    32 | #  include <quadmath.h>
#12 364.6       |            ^~~~~~~~~~~~
#12 364.6 compilation terminated.
#12 364.6 In file included from libs/charconv/build/../src/from_chars_float_impl.hpp:9,
#12 364.6                  from libs/charconv/build/../src/from_chars.cpp:14:
#12 364.6 ./boost/charconv/detail/config.hpp:32:12: fatal error: quadmath.h: No such file or directory
#12 364.6    32 | #  include <quadmath.h>
#12 364.6       |            ^~~~~~~~~~~~
#12 364.6 compilation terminated.
#12 372.5 ...failed updating 0 target...
#12 372.5 boost/1.85.0: ERROR:
#12 372.5 Package '3d05c3311f889e9a4efe75ff8d85497ffa81de4a' build failed
#12 372.5 ERROR: boost/1.85.0: Error in build() method, line 1167
#12 372.5   ConanException: Error 1 while executing
#12 372.6 make[1]: *** [Makefile:257: conan_build] Error 1
#12 372.6 make: *** [Makefile:315: release] Error 2
```
（日志中先出现的前置征兆：`#12 318.3 - GCC libquadmath and __float128 support : no [2]`，即 boost 配置检测阶段已判定 quadmath 不可用，随后 charconv 仍包含 `<quadmath.h>` 导致编译中止。）

### 根因定位
- 失败位置: Dockerfile:21（`RUN bash scripts/install-bolt-deps.sh && conan profile detect && make release && make export_release`）
- 失败原因: bolt 通过 conan 从源码构建 boost 1.85.0，`boost/charconv` 的 `config.hpp` 需要 `quadmath.h`；该头文件由 `libquadmath-devel` 提供，而新 Dockerfile 的 `dnf install` 只安装了 `gcc gcc-c++`（未包含 `libquadmath-devel`），导致编译期找不到 `quadmath.h`，boost 构建失败，连带 `conan_build` / `make release` 退出码 2。

### 与 PR 变更的关联
本 PR 新增文件 `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile`，其依赖安装行为：
```dockerfile
RUN dnf install -y \
      git gcc gcc-c++ make cmake ninja-build patch \
      libstdc++-static glibc-static \
      curl \
      python3-pip && \
    dnf clean all
```
安装列表中缺少提供 `quadmath.h` 的 `libquadmath-devel`。失败发生在新 Dockerfile 的第 21 行 `make release`（conan 构建 boost）。因此该失败**与本次 PR 新增的 Dockerfile 直接相关**（新增文件即触发构建的镜像），不是历史遗留问题。

## 修复方向

### 方向 1（置信度: 高）
在 Dockerfile 的 `dnf install` 依赖列表中补充 `libquadmath-devel`（openEuler 中提供 `quadmath.h` 的开发包），使 boost/charconv 能正常编译，从而修复 conan 构建 boost 1.85.0 的失败。

### 方向 2（可选，置信度: 中）
若上游 `scripts/install-bolt-deps.sh` 或 conan 对 openEuler 有额外系统依赖约定，可确认其实际安装的依赖集合，并据此补齐缺失的开发包（`libquadmath-devel` 优先级最高）。也可考虑在构建中显式关闭 boost/charconv 的 float128 支持（如通过 conan/编译选项），以避免对 `quadmath.h` 的依赖，但该改动涉及上游构建脚本，风险高于方向 1。

## 需要进一步确认的点
1. 确认 openEuler 24.03-LTS-SP4 仓库中 `libquadmath-devel` 包的实际可用性与包名（应直接提供 `/usr/lib64/gcc/.../include/quadmath.h`），以及 `gcc-c++` 是否随版本变化已隐含该包。
2. 确认上游 `bytedance/bolt` 的 `scripts/install-bolt-deps.sh` 是否本应安装该依赖；若是脚本内部已尝试安装但被跳过/失败，需查看该脚本在 openEuler 下的分支逻辑。
3. 确认是否存在 boost 1.85 charconv 在无 quadmath 环境下的官方规避方式（如宏定义/选项），以判断方向 1 与方向 2 哪个更符合上游约定。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
不适用：本失败不涉及对第三方/上游源文件的正则 patch。

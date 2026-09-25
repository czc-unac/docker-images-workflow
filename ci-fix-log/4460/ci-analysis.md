# CI 失败分析报告

## 基本信息
- PR: #4460 — 【自动升级】bolt容器镜像升级至2d01261版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式10（缺少构建依赖，变体：缺少编译头文件而非 cmake configure 阶段报错）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#12 413.1     - GCC libquadmath and __float128 support : no [2]
...
#12 459.4 In file included from ./boost/charconv/detail/emulated128.hpp:11,
#12 459.4                  from ./boost/charconv/detail/apply_sign.hpp:9,
#12 459.4                  from libs/charconv/build/../src/to_chars_float_impl.hpp:10,
#12 459.4                  from libs/charconv/build/../src/to_chars.cpp:7:
#12 459.4 ./boost/charconv/detail/config.hpp:32:12: fatal error: quadmath.h: No such file or directory
#12 459.4    32 | #  include <quadmath.h>
#12 459.4       |            ^~~~~~~~~~~~
#12 459.4 compilation terminated.
...
#12 467.7 boost/1.85.0: ERROR:
#12 467.7 Package '3d05c3311f889e9a4efe75ff8d85497ffa81de4a' build failed
#12 467.7 ERROR: boost/1.85.0: Error in build() method, line 1167
#12 467.7 	ConanException: Error 1 while executing
#12 467.8 make[1]: *** [Makefile:257: conan_build] Error 1
#12 467.8 make: *** [Makefile:315: release] Error 2
ERROR: failed to solve: ... exit code: 2
```

### 根因定位
- 失败位置: `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile:21`（`RUN bash scripts/install-bolt-deps.sh && conan profile detect && make release && make export_release`）
- 失败原因: 构建依赖 boost/1.85.0 的 charconv 组件编译时找不到 `quadmath.h` 头文件（该头文件由 GCC 的 libquadmath 开发包提供），而 Dockerfile 的 `dnf install` 只安装了 `gcc gcc-c++ make cmake ninja-build patch libstdc++-static glibc-static curl python3-pip`，遗漏了提供 `quadmath.h` 的 `libquadmath-devel`。boost b2 配置阶段已检测到 `GCC libquadmath and __float128 support : no`，随后 charconv 头文件仍尝试 `#include <quadmath.h>`，编译直接终止，导致 conan 构建 boost 失败，进而 `make release` 返回 Error 2。

### 与 PR 变更的关联
本次 PR 新增了 `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile`，构建路径为 `/workspace/bolt` 下执行 `make release`，由 bolt 的 conan 依赖解析出 boost/1.85.0 源码构建。失败发生在该新增 Dockerfile 的构建步骤中，与新增文件直接相关（即新版本 bolt + 新基础镜像 sp4 的依赖组合缺少 `libquadmath-devel`）。README.md / image-info.yml / meta.yml 的元数据改动不影响构建本身。

## 修复方向

### 方向 1（置信度: 高）
在新增 Dockerfile 的第一个 `dnf install` 步骤中补充 `libquadmath-devel`（openEuler 中提供 `quadmath.h` 的开发包），使 boost/1.85.0 charconv 组件能正常 `#include <quadmath.h>`。同步确认 `install-bolt-deps.sh` 内部是否已安装该包，若脚本已安装则问题定位为脚本在 sp4 上的包名/可用性差异。

### 方向 2（置信度: 中）
若 openEuler 24.03-LTS-SP4 仓库中 `libquadmath-devel` 不可用，则在 conan 构建层规避：为 boost 关闭 charconv 组件或关闭 float128 支持（如 conan 配置中的 `without_charconv=True` / 禁用 quadmath 探测），使 charconv 走通用实现路径。此方向需修改 bolt 的 conan 配置或构建参数，影响面较大，应优先采用方向 1。

## 需要进一步确认的点
- 确认 openEuler 24.03-LTS-SP4 官方仓库中 `libquadmath-devel` 的确切包名及可用性（`quadmath.h` 也可能由 `gcc-toolset` 或 `libquadmath` 相关包提供）。
- 确认 bolt 仓库 `2d01261` 版本 `scripts/install-bolt-deps.sh` 的实际内容，核对其是否本应安装 libquadmath 开发包以及是否依赖特定基础镜像。
- 确认参考版本 `6b54e46/24.03-lts-sp3` 的 Dockerfile 是否同样缺失该依赖（若 sp3 能成功，说明差异在 sp4 基础镜像的默认包集合或 boost 版本解析）。
- 确认 boost/1.85.0 是 bolt 仓库固定锁定版本还是 conan 运行时解析结果（判断是否还需处理上游依赖升级带来的兼容问题）。

## 修复验证要求
不涉及正则 patch 外部源文件，无强制上游验证要求。建议 code-fixer 在提交前于本地/CI 复现 `dnf install libquadmath-devel` 后 boost/1.85.0 能否通过 charconv 编译（即 `quadmath.h` 可被找到）。因修复方向置信度为高，但仍需以实际包可用性为准。

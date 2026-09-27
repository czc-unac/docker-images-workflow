# CI 失败分析报告

## 基本信息
- PR: #4625 — 【自动升级】bolt容器镜像升级至2d01261版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式10（缺少构建依赖）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#12 338.8     - GCC libquadmath and __float128 support : no [2]
#12 385.2 ./boost/charconv/detail/config.hpp:32:12: fatal error: quadmath.h: No such file or directory
#12 385.2    32 | #  include <quadmath.h>
#12 385.2       |            ^~~~~~~~~~~~
#12 385.2 compilation terminated.
#12 393.4 boost/1.85.0: ERROR:
#12 393.4 Package '3d05c3311f889e9a4efe75ff8d85497ffa81de4a' build failed
#12 393.4 ERROR: boost/1.85.0: Error in build() method, line 1167
#12 393.4 	ConanException: Error 1 while executing
#12 393.5 make[1]: *** [Makefile:257: conan_build] Error 1
#12 393.5 make: *** [Makefile:315: release] Error 2
```

### 根因定位
- 失败位置: `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile:21-24`（`RUN bash scripts/install-bolt-deps.sh && conan profile detect && make release && make export_release`），实际编译报错点在 conan 拉取的 `boost/1.85.0` 源码 `boost/charconv/detail/config.hpp:32`。
- 失败原因: 新镜像构建过程中，conan 以源码方式编译 `boost/1.85.0` 的 `charconv` 组件，该组件在编译时无条件包含 `<quadmath.h>`（Boost 1.85 charconv 的 float 后端）。构建环境（openEuler 24.03-LTS-SP4 基础镜像）只安装了 `gcc gcc-c++`，未安装提供 `quadmath.h` 的 `libquadmath-devel`，导致头文件缺失、编译中止，boost 包构建失败，进而 `make release` 返回 exit code 2。

### 与 PR 变更的关联
本 PR 新增 `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile`（新版本、新基础镜像 24.03-lts-sp4），其 `dnf install` 步骤只安装了 `git gcc gcc-c++ make cmake ninja-build patch libstdc++-static glibc-static curl python3-pip`，未包含 `libquadmath-devel`。构建日志中的失败发生在该 Dockerfile 第 21-24 行的 RUN 指令内（`Dockerfile:21` 定位），因此失败由本次新增文件直接触发。
（`README.md`、`doc/image-info.yml`、`meta.yml` 的改动为文档/元数据登记，与本次构建失败无直接关系。）

## 修复方向

### 方向 1（置信度: 高）
在 `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile` 首个 `dnf install` 步骤中补充提供 `quadmath.h` 的开发包（openEuler 24.03 中为 `libquadmath-devel`，属于 GCC 运行时/开发组件），使 boost charconv 的编译能找到 `quadmath.h`。

### 方向 2（置信度: 中）
若补齐系统包后仍失败，则在 conan/boost 构建配置层面禁用 charconv 的 quadmath 路径（例如通过 conan boost 选项关闭 `without_charconv` 或在 b2 配置中不启用 charconv），避免编译依赖 quadmath 的组件。该方向需先确认 bolt 上游是否确实需要 charconv。

## 需要进一步确认的点
1. 确认 openEuler 24.03-LTS-SP4 仓库中提供 `quadmath.h` 的确切包名（预期为 `libquadmath-devel`；需与实际 `dnf` 仓库核对）。
2. 对照同目录旧版本 `Bigdata/bolt/6b54e46/24.03-lts-sp3/Dockerfile` 的 `dnf install` 列表，确认其是否安装了 quadmath 相关包或使用了不同的 boost 版本，以解释旧版本为何未触发该问题。
3. 确认该镜像的目标构建架构（meta.yml 是否声明 `arch`），本日志显示为 `architecture=x86` / `x86_64`，故非架构不匹配问题。

## 修复验证要求
本修复方向不涉及通过正则 patch 第三方/上游源文件（仅在 Dockerfile 层补充系统依赖包），因此无需上游正则匹配验证。但 code-fixer 在提交前必须：核对 openEuler 24.03-LTS-SP4 仓库中 `quadmath.h` 的实际提供包名确实为拟新增的包名，避免引入不存在的包导致 `dnf install` 新的失败。

# CI 失败分析报告

## 基本信息
- PR: #4529 — 【自动升级】tvm容器镜像升级至0.27.0版本.
- 失败类型: build-error
- 置信度: 中
- 知识库匹配: 新模式
- 新模式标题: 缺少标准库头文件
- 新模式症状关键词: stdlib.h, No such file or directory, include_next, cstdlib, gcc-c++, glibc-devel

## 根因分析

### 直接错误
```
#10 1.445 [  2%] Building CXX object CMakeFiles/tvm_objs.dir/src/ir/attrs.cc.o
#10 1.513 In file included from /usr/include/c++/12/ext/string_conversions.h:41,
#10 1.513                  from /usr/include/c++/12/bits/basic_string.h:3968,
#10 1.513                  from /usr/include/c++/12/string:53,
#10 1.513                  from /tvm/3rdparty/tvm-ffi/include/tvm/ffi/type_traits.h:30,
#10 1.513                  from ... /tvm/src/ir/attrs.cc:23:
#10 1.513 /usr/include/c++/12/cstdlib:75:15: fatal error: stdlib.h: No such file or directory
#10 1.513    75 | #include_next <stdlib.h>
#10 1.513       |               ^~~~~~~~~~
#10 1.513 compilation terminated.
#10 1.516 gmake[2]: *** [CMakeFiles/tvm_objs.dir/build.make:79: CMakeFiles/tvm_objs.dir/src/ir/attrs.cc.o] Error 1
#10 1.516 gmake[1]: *** [CMakeFiles/Makefile2:441: CMakeFiles/tvm_objs.dir/all] Error 2
#10 55.83 gmake: *** [Makefile:136: all] Error 2
#10 ERROR: process "... cmake .. && cmake --build . --parallel $(nproc)" did not complete successfully: exit code: 2
```

### 根因定位
- 失败位置: `AI/tvm/0.27.0/24.03-lts-sp4/Dockerfile:15-20`（`RUN cp /tvm/cmake/config.cmake . && ... cmake .. && cmake --build . --parallel $(nproc)` 步骤），编译对象 `CMakeFiles/tvm_objs.dir/src/ir/attrs.cc.o`（build.make:79）；另有 `src/runtime/vm/executable.cc` 同样触发。
- 失败原因: C++ 标准库头 `/usr/include/c++/12/cstdlib:75` 执行 `#include_next <stdlib.h>` 时找不到 C 标准库头文件 `stdlib.h`，编译器直接 `fatal error` 终止。这是**全局的 C 库开发头缺失**问题（整个 `<string>`/`<optional>` 链都因此失败），并非 TVM 源码本身的编译错误。日志显示该 Docker 阶段编译首个对象即失败，所有并行编译目标（tvm_objs / tvm_runtime_objs）均受影响。

### 与 PR 变更的关联
直接相关。本 PR 新增 `AI/tvm/0.27.0/24.03-lts-sp4/Dockerfile`，其依赖安装步骤为：
```
RUN yum install -y git cmake make gcc-c++ libxml2-devel llvm-devel zlib-devel python3-devel python3-pip
```
该步骤未显式声明 glibc 开发头文件包（`glibc-devel` / `glibc-headers`）。日志的 `Installed:` 列表中出现了 `glibc-devel-2.38-107.oe2403sp4`，但**未出现提供 `/usr/include/stdlib.h` 的 `glibc-headers`**（同时 `kernel-headers` 已安装）。即安装清单在头文件层面不完整，导致 0.27.0 的新构建在编译阶段找不到 `stdlib.h`。

### 非根因说明（日志中的干扰项，均非致命）
- `CMake Warning at FindZ3.cmake:116 — Failed to determine Z3 library version, defaulting to 0.0.0.` → 仅为告警，随后正常 `-- Build without Z3 SMT solver support`。
- `Could NOT find GTest` / `Could NOT find FFI` → 可选组件缺失的常规提示。
- `UndefinedVar: Usage of undefined variable '$PYTHONPATH' (line 28)` → BuildKit lint 警告（`ENV PYTHONPATH=/tvm/python:$PYTHONPATH` 自引用未定义变量），非构建失败原因，但建议顺手按 `${PYTHONPATH:-}` 处理。
- `perl-*`、`Verifying`/`Installed` 大段输出 → 属 `#7` yum 安装步骤的正常回显。

## 修复方向

### 方向 1（置信度: 中）
在 Dockerfile 的 `yum install` 步骤中**显式补齐 glibc 开发头文件包**（如 `glibc-devel`，必要时同时确认 `glibc-headers`），确保 `/usr/include/stdlib.h` 存在于构建镜像中。这是最直接的修复路径：当前安装清单缺少对标准 C 库头文件的显式依赖声明。

### 方向 2（置信度: 中）
若确认 openEuler 24.03-LTS-SP4 基础镜像/仓库中头文件由独立包提供，则应在同一个 `RUN yum install` 中显式安装该头文件包；并核对基础镜像是否为精简版导致 `/usr/include` 不完整。可同时对该阶段使用完整构建工具链基线（如 `gcc gcc-c++ make cmake glibc-devel`）以避免隐式依赖缺失。

### 方向 3（可选，低-中）
对 `$PYTHONPATH` 自引用警告做无害化处理（`${PYTHONPATH:-}`），消除 BuildKit UndefinedVar 告警（与本次失败无直接因果）。

## 需要进一步确认的点
1. `stdlib.h` 在 openEuler 24.03-LTS-SP4 中由哪个 RPM 提供（`glibc-headers` 还是已并入 `glibc-devel`），以及为何 yum 事务中 `glibc-devel` 已安装却不见 `glibc-headers` / `/usr/include/stdlib.h`。
2. 对比同仓库 `AI/tvm/0.24.0/24.03-lts-sp4/Dockerfile` 的安装包清单与基础镜像来源，确认为何 0.24.0 可构建而 0.27.0 失败（是包清单差异还是基础镜像/仓库变化）。
3. 基础镜像 `openeuler/openeuler:24.03-lts-sp4` 是否为精简镜像、其 `/usr/include` 初始内容及 repo 配置是否完整。
4. 日志未显示 `#7` 安装步骤之后该层是否实际保留了 `/usr/include/stdlib.h`（日志不足以直接验证）。

## 修复验证要求
本失败置信度为"中"，code-fixer 在提交修复前必须执行以下验证，不得假设方向 1 一定正确：
1. 启动/进入与失败构建相同的基础镜像（`openeuler/openeuler:24.03-lts-sp4`），执行 `rpm -qf /usr/include/stdlib.h` 与 `ls -l /usr/include/stdlib.h`，确认头文件是否存在及其归属包。
2. 按方向 1 修改后，在容器内执行 `yum install` 并**再次验证 `/usr/include/stdlib.h` 真实存在**，再触发构建；同时确认 gcc 编译一个最小 `#include <cstdlib>` 程序不再报 `stdlib.h: No such file or directory`。
3. 确认修复后 `cmake --build . --parallel $(nproc)` 能越过 `src/ir/attrs.cc`、`src/runtime/vm/executable.cc` 的编译（即不再在 1.5s 内以相同 `fatal error` 失败），而不只是消除了 `yum` 阶段的差异。

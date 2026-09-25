# CI 失败分析报告

## 基本信息
- PR: #4445 — 【自动升级】tvm容器镜像升级至0.26.0版本.
- 失败类型: build-error
- 置信度: 中
- 知识库匹配: 新模式
- 新模式标题: stdlib头文件缺失
- 新模式症状关键词: stdlib.h, No such file or directory, cstdlib, include_next, LLVM_INCLUDE_DIRS, tvm_objs

## 根因分析

### 直接错误
```
#10 1.416 [  3%] Building CXX object CMakeFiles/tvm_objs.dir/src/arith/analyzer.cc.o
#10 1.539 In file included from /usr/include/c++/12/ext/string_conversions.h:41,
#10 1.539                  from /usr/include/c++/12/bits/basic_string.h:3968,
#10 1.539                  from /usr/include/c++/12/string:53,
#10 1.539                  from /tvm/3rdparty/tvm-ffi/include/tvm/ffi/type_traits.h:30,
#10 1.539                  from ... /tvm/include/tvm/arith/analyzer.h:27,
#10 1.539                  from /tvm/src/arith/analyzer.cc:23:
#10 1.539 /usr/include/c++/12/cstdlib:75:15: fatal error: stdlib.h: No such file or directory
#10 1.539    75 | #include_next <stdlib.h>
#10 1.539       |               ^~~~~~~~~~
#10 1.539 compilation terminated.
#10 1.541 gmake[2]: *** [CMakeFiles/tvm_objs.dir/build.make:79: CMakeFiles/tvm_objs.dir/src/arith/analyzer.cc.o] Error 1
#10 1.541 gmake[1]: *** [CMakeFiles/Makefile2:441: CMakeFiles/tvm_objs.dir/all] Error 2
#10 48.72 gmake: *** [Makefile:136: all] Error 2
#10 ERROR: process "... cmake --build . --parallel $(nproc)" did not complete successfully: exit code: 2
```

### 根因定位
- 失败位置: `/tvm/src/arith/analyzer.cc:23`（包含链：`/usr/include/c++/12/string:53` → `bits/basic_string.h:3968` → `ext/string_conversions.h:41` → `cstdlib:75`）
- 失败原因: libstdc++ 头文件 `<cstdlib>` 内的 `#include_next <stdlib.h>` 无法定位系统 C 标准库头文件，编译中止，导致 TVM 核心目标 `tvm_objs` 失败、整体 `cmake --build` 返回 exit code 2。

**关键证据（说明不是“缺包”）**：
1. 日志中 `yum install` 已完成，`glibc-devel-2.38-107.oe2403sp4` 已安装（见 “Installed” 列表）。
2. 同一个 Dockerfile 的编译过程中，不依赖 LLVM 的目标 `tvm_runtime_objs` 全部编译成功（包括 `src/runtime/vm/executable.cc`，它同样会经 `<string>` 间接包含 `<cstdlib>`，见 `#10 29.57` 的告警链）。
3. 只有依赖 LLVM 的目标 `tvm_objs` 的**第一个**编译单元 `src/arith/analyzer.cc` 立即失败。
4. cmake 配置阶段明确输出 `-- Found LLVM_INCLUDE_DIRS=/usr/include`。

综合以上，最可能的原因是：openEuler 的 `llvm-devel` 把 LLVM 头文件安装到 `/usr/include`，TVM 0.26.0 在启用 `USE_LLVM` 时将该目录以普通 `-I`（非 system）方式加入 `tvm_objs` 的编译命令。GCC 一旦以 `-I/usr/include` 注入系统目录，会破坏 libstdc++ 的 `#include_next` 搜索链，于是 `<cstdlib>` 的 `#include_next <stdlib.h>` 报 “No such file or directory”。这正好解释了“含 LLVM 的目标失败、纯 runtime 目标成功”的差异。

### 与 PR 变更的关联
本次 PR 新增 `AI/tvm/0.26.0/24.03-lts-sp4/Dockerfile`，其中安装 `llvm-devel` 并在 `config.cmake` 中追加 `set(USE_LLVM ON)`，随后执行 `cmake .. && cmake --build .`。失败发生在该 Dockerfile 第 15-20 行的构建步骤内，属于本次新增内容直接触发的构建错误。

## 修复方向

### 方向 1（置信度: 中）
避免让 `/usr/include` 以非 system 方式进入 `tvm_objs` 的编译命令：在 `git clone` TVM 源码后、`cmake` 之前，对 TVM 0.26.0 处理 LLVM include 的 CMake 模块做修正，使 LLVM 的 include 目录以 system 方式引入，或在 LLVM include 列表中剔除 `/usr/include`（因为该目录本就是 GCC 默认系统目录，无需重复加入）。这是与日志证据（仅 LLVM 目标失败 + `LLVM_INCLUDE_DIRS=/usr/include`）最吻合的方向。

### 方向 2（置信度: 中）
若方向 1 验证后发现 TVM 源码中并不存在显式注入 `/usr/include` 的逻辑，则退回到“构建依赖/头文件”方向排查：显式在 `yum install` 中补全 C/C++ 开发头文件包（`glibc-headers` / `glibc-devel`、`libstdc++-devel`），并确认基础镜像内 `glibc` 运行时与 `glibc-devel` 版本一致，排除头文件包缺失或版本错配。注意：现有日志显示 `glibc-devel` 已安装且 runtime 目标可正常包含 `<cstdlib>`，因此该方向可能性低于方向 1。

### 方向 3（置信度: 低）
若上述均不成立，可考虑调整 LLVM 的引入方式（例如让 `LLVM_INCLUDE_DIRS` 指向非系统目录的 LLVM 头文件位置，或改用与 openEuler 默认头文件布局不冲突的 LLVM 配置），以避免与 `/usr/include` 冲突。

## 需要进一步确认的点
1. TVM 0.26.0 源码中 `cmake/modules/LLVM.cmake`（或等价模块）如何把 `LLVM_INCLUDE_DIRS` 加入编译：是否为非 system 的 `include_directories(...)`，以及 TVM 0.26.0 相对 0.24.0 是否改动了此处逻辑。
2. openEuler `llvm-devel` 提供的 `llvm-config --includedir` 实际返回值是否就是 `/usr/include`（日志已显示 `LLVM_INCLUDE_DIRS=/usr/include`，可与上游确认）。
3. 旧版本 `AI/tvm/0.24.0/24.03-lts-sp4/Dockerfile` 是如何规避同一问题的（可对比其 `USE_LLVM`/LLVM include 处理），以判断本次升级引入的差异。
4. 确认 `tvm_runtime_objs` 与 `tvm_objs` 编译命令的实际差异（`compile_commands.json` 或 `flags.make`），以最终坐实“仅 LLVM 目标带有 `/usr/include` 的 `-I`”。

## 修复验证要求
本修复方向 1 涉及在 `git clone` 得到的外部源码（apache/tvm，`ARG VERSION=0.26.0`，tag `v0.26.0`）上做正则/patch 修改，因此：
- code-fixer 在提交前，必须从 apache/tvm `v0.26.0` 拉取对应的 CMake 模块文件（如 `cmake/modules/LLVM.cmake`），核对其真实内容与目标字符串，验证新正则/替换确实能匹配后再提交，不能假设文件内容与旧版一致。
- 修复后需在 openEuler 24.03-LTS-SP4 + `llvm-devel` 环境下实际执行 `cmake .. && cmake --build .` 验证 `tvm_objs` 的 `src/arith/analyzer.cc` 能通过编译。
- 若方向 1 未能复现/命中，必须回退到方向 2 并验证补齐的包名在 openEuler 24.03-LTS-SP4 仓库中真实存在。

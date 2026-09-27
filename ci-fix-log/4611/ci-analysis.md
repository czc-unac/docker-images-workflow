# CI 失败分析报告

## 基本信息
- PR: #4611 — 【自动升级】tvm容器镜像升级至0.27.0版本.
- 失败类型: build-error
- 置信度: 中
- 知识库匹配: 新模式（概念上接近 模式10「缺少构建依赖」，但报错形态不同）
- 新模式标题: 编译缺失stdlib.h
- 新模式症状关键词: fatal error, stdlib.h, No such file or directory, cstdlib, include_next, glibc-devel, cmake --build

> 前置检查说明：`ci.logs` 末尾为 `Build step 'Execute shell' marked build as failure` 与 `Finished: FAILURE`，未出现 `Finished: SUCCESS` / `Build successful`，故属于真实构建失败，继续分析。

## 根因分析

### 直接错误
```
#10 1.418 [  0%] Building CXX object CMakeFiles/tvm_objs.dir/src/ir/attrs.cc.o
#10 1.526 In file included from /usr/include/c++/12/ext/string_conversions.h:41,
#10 1.526                  from /usr/include/c++/12/bits/basic_string.h:3968,
#10 1.526                  from /usr/include/c++/12/string:53,
#10 1.526                  from /tvm/3rdparty/tvm-ffi/include/tvm/ffi/type_traits.h:30,
...
#10 1.526                  from /tvm/src/ir/attrs.cc:23:
#10 1.526 /usr/include/c++/12/cstdlib:75:15: fatal error: stdlib.h: No such file or directory
#10 1.526    75 | #include_next <stdlib.h>
#10 1.526 compilation terminated.
#10 1.536 gmake[2]: *** [CMakeFiles/tvm_objs.dir/build.make:79: CMakeFiles/tvm_objs.dir/src/ir/attrs.cc.o] Error 1
#10 1.536 gmake[1]: *** [CMakeFiles/Makefile2:441: CMakeFiles/tvm_objs.dir/all] Error 2
#10 52.49 gmake: *** [Makefile:136: all] Error 2
#10 ERROR: process "... cp /tvm/cmake/config.cmake . && ... cmake .. && cmake --build . --parallel $(nproc)" did not complete successfully: exit code: 2
```

### 根因定位
- 失败位置: `AI/tvm/0.27.0/24.03-lts-sp4/Dockerfile:15-20`（`RUN cp ... cmake .. && cmake --build . --parallel $(nproc)` 步骤），触发文件为上游源码 `/tvm/src/ir/attrs.cc:23`，实际报错发生在 `/usr/include/c++/12/cstdlib:75`。
- 失败原因: C++ 标准头 `cstdlib` 通过 `#include_next <stdlib.h>` 找不到 C 标准库头 `stdlib.h`，编译终止，`cmake --build` 返回 exit code 2。此类报错的典型成因是容器内缺少提供 `/usr/include/stdlib.h` 的开发包头（openEuler 中通常由 `glibc-headers` / `glibc-devel` 提供），即 Dockerfile 的 `yum install` 依赖清单不完整（详见"需要进一步确认的点"）。

### 与 PR 变更的关联
- PR 新增了 `AI/tvm/0.27.0/24.03-lts-sp4/Dockerfile`（并同步 README/image-info.yml/meta.yml）。CI 失败的 `#10` 步骤正是该新增 Dockerfile 中的 cmake 编译步骤，因此失败由本次 PR 直接引入。
- Dockerfile 的 `yum install` 清单为：`git cmake make gcc-c++ libxml2-devel llvm-devel zlib-devel python3-devel python3-pip`。清单中虽通过 `gcc-c++` 间接引入 `glibc-devel`，但日志显示编译 `tvm_objs` 目标时仍找不到 `stdlib.h`。
- 次要（非致命）问题：日志末尾 lint 提示 `UndefinedVar: Usage of undefined variable '$PYTHONPATH' (line 28)`，对应 Dockerfile 的 `ENV PYTHONPATH=/tvm/python:$PYTHONPATH` 自引用未定义变量（与知识库模式20同源），不是本次失败根因。

## 修复方向

### 方向 1（置信度: 中）
在 Dockerfile 的 `yum install` 步骤中补齐 C 标准库开发头相关包（openEuler 中通常为 `glibc-headers`，可一并显式声明 `glibc-devel`、必要时 `kernel-headers`），确保 `/usr/include/stdlib.h` 存在。可对照同仓库 `AI/tvm/0.24.0/24.03-lts-sp4/Dockerfile` 的依赖清单，取其可用依赖集作为基线。

### 方向 2（置信度: 低）
若补齐依赖后仍报同样错误，则需排查 `tvm_objs` 目标相对 `tvm_runtime_objs` 的编译参数差异（如自定义 include 顺序 / `-nostdinc` 类参数导致 `#include_next` 跳过 `/usr/include`），或检查 openEuler 24.03-LTS-SP4 基础镜像内 glibc 头包与 `glibc-devel` 版本的匹配情况。

### 附带（非根因，可选清理）
将 `ENV PYTHONPATH=/tvm/python:$PYTHONPATH` 改为使用默认值语法（如 `${PYTHONPATH:-}`）可消除 BuildKit `UndefinedVar` 警告。

## 需要进一步确认的点
1. 对比 `AI/tvm/0.24.0/24.03-lts-sp4/Dockerfile` 与新版 0.27.0 Dockerfile 的 `yum install` 依赖清单差异，确认旧版是否显式安装了 `glibc-headers` / `kernel-headers`（这是判断"缺包"假设最直接的证据）。
2. 确认 openEuler 24.03-LTS-SP4 中提供 `/usr/include/stdlib.h` 的具体 RPM（预期为 `glibc-headers`），以及 `glibc-devel` 是否会自动拉起该包；日志"Installed"清单中列有 `glibc-devel-2.38-107.oe2403sp4` 但未列出 `glibc-headers`。
3. 日志内部存在不一致需要解释：同一 `#10` 步骤中 `tvm_runtime_objs` 目标（如使用 `<string>` 的 `/tvm/src/runtime/vm/executable.cc`、`workspace_pool.cc`）编译成功，仅 `tvm_objs` 目标的 `/tvm/src/ir/attrs.cc` 报 `stdlib.h` 缺失。需确认两个目标的编译参数/包含路径差异，以排除"并非全局缺头文件"的其它解释。
4. 需要 aarch64（arm64）下游 job 的日志，确认该问题是否两架构均复现（若仅单架构，则更偏向架构相关的依赖/参数问题）。

## 修复验证要求
本修复不涉及对第三方/上游源文件的正则 patch（不属于模板中"正则 patch 外部源文件"场景）。
但置信度为"中"，code-fixer 在提交前必须执行以下验证，不得直接假设修复方向正确：
1. 在目标基础镜像 `openeuler/openeuler:24.03-lts-sp4` 中确认 `/usr/include/stdlib.h` 的缺失状态，并确认补充 `glibc-headers`（或等价包）后该文件存在。
2. 以 `git clone --recursive -b v0.27.0 https://github.com/apache/tvm.git` 获取的 TVM 0.27.0 源码为准，在补充依赖后的容器内实际执行 Dockerfile 第 15-20 行的 cmake 编译步骤，确认 `src/ir/attrs.cc` 能通过编译。
3. 若修复方向采用"补齐依赖"，需对照 `0.24.0` 旧版 Dockerfile 已验证可用的依赖清单，避免引入与新版不兼容的包。

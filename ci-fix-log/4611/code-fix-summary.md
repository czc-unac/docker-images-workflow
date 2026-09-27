# 修复摘要

## 修复的问题
恢复 TVM 构建时对上游 `cmake/modules/LLVM.cmake` 的 `-isystem` → `-I` 补丁，修复在 openEuler 24.03-LTS-SP4 上编译 TVM 0.27.0 时 `cstdlib:75: fatal error: stdlib.h: No such file or directory` 的构建失败。

## 修改的文件
- `AI/tvm/0.27.0/24.03-lts-sp4/Dockerfile`: 在 `git clone` 后追加对上游 `LLVM.cmake` 的 sed 补丁与校验，将 LLVM 头文件路径由 `-isystem` 改为 `-I`。

## 修复逻辑
CI 分析报告推测“缺少 glibc-headers/glibc-devel 导致 `/usr/include/stdlib.h` 缺失”，但该推测经实测被否定：

1. 拉取基础镜像 `openeuler/openeuler:24.03-lts-sp4`，安装 Dockerfile 中的依赖（`gcc-c++` 等）后，`/usr/include/stdlib.h` **存在**，且由 `glibc-devel-2.38-107.oe2403sp4` 提供；单独的 `#include <cstdlib>` 编译通过。故“缺包”不是根因。
2. 真正根因：openEuler 的 `llvm-devel` 将头文件安装在 `/usr/include`，`llvm-config --includedir` 返回 `/usr/include`。TVM 0.27.0 的 `cmake/modules/LLVM.cmake` 会把该路径以 `-isystem /usr/include` 加入编译选项，重复了默认系统头目录，导致 GCC 在 `/usr/include/c++/12/cstdlib:75` 执行 `#include_next <stdlib.h>` 时无法找到 `/usr/include/stdlib.h`。
3. 本次 PR 在升级 0.26.0 → 0.27.0 时**删除了**旧版 Dockerfile 中已有的规避补丁（`sed -i 's/"-isystem"/"-I"/' /tvm/cmake/modules/LLVM.cmake`），因此才引入失败。本修复即恢复该补丁（`pr.changed_files` 内文件，改动最小）。

### 验证结果（已实际执行）
- 已从上游 `apache/tvm` tag `v0.27.0` 获取实际源文件 `cmake/modules/LLVM.cmake`，确认其中存在字面量 `"-isystem"`（`list(APPEND TVM_LLVM_INCLUDE_FLAGS "-isystem" ...)`），sed 能匹配；执行后 `grep -q 'TVM_LLVM_INCLUDE_FLAGS "-I"'` 校验通过。
- 在基础镜像 `openeuler/openeuler:24.03-lts-sp4` 中复现：
  - 未打补丁：编译 `src/ir/attrs.cc` 报与 CI 完全一致的 `cstdlib:75: fatal error: stdlib.h: No such file or directory`，`make` 退出码 2。
  - 打补丁后：同一对象文件编译成功（`make` 退出码 0），编译命令中 `-isystem /usr/include` 变为 `-I /usr/include`。

## 潜在风险
无。该改动仅恢复仓库此前 0.24.0/0.26.0 版本已采用且验证可用的规避手段，作用域限定为 TVM 源码构建期的 LLVM 头文件包含方式（变更为普通包含路径，不改变链接库与功能）。分析报告中提到的 `ENV PYTHONPATH=/tvm/python:$PYTHONPATH` 引发的 `UndefinedVar` 告警为非致命、非本次失败根因，按最小化原则未做改动。
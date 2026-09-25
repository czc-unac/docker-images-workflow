# 修复摘要

## 修复的问题
修复 TVM 0.26.0 在 openEuler 24.03-LTS-SP4 上因 `-isystem /usr/include` 破坏 libstdc++ `#include_next <stdlib.h>` 搜索链，导致 `tvm_objs` 编译 `src/arith/analyzer.cc` 时 `stdlib.h: No such file or directory`、镜像构建失败的问题。

## 修改的文件
- `AI/tvm/0.26.0/24.03-lts-sp4/Dockerfile`: 在 `git clone` TVM 源码后、`cmake` 之前，用 `sed` 将 `cmake/modules/LLVM.cmake` 中唯一一处带引号的 `"-isystem"` 改为 `"-I"`，并加 `grep -q` 断言确保 patch 生效。

## 修复逻辑
- 根因（与分析报告方向 1 一致，并已实测复现）：openEuler 的 `llvm-devel` 将 LLVM 头文件安装到 `/usr/include`，`llvm-config --includedir` 返回 `/usr/include`，因此 TVM 的 `LLVM_INCLUDE_DIRS=/usr/include`。
- TVM 0.24.0 的 `cmake/modules/LLVM.cmake` 只使用 `include_directories(SYSTEM ${LLVM_INCLUDE_DIRS})`；CMake 会将 `/usr/include` 识别为隐式系统目录并从编译命令中过滤（实测 `flags.make` 中不出现该 `-isystem`），因此旧版构建正常。
- TVM 0.26.0 新增 `tvm_llvm_header` INTERFACE 目标，并在其 `target_compile_options(... INTERFACE "-isystem" "${__llvm_include_dir}")` 中显式注入 `-isystem /usr/include`。该目标通过 `target_link_libraries(tvm_objs PUBLIC tvm_llvm_header)` 只作用于 `tvm_objs`（编译器目标），而 `tvm_runtime_objs` 仅链接 `tvm_ffi_header`，不会带入该 flag——这与“仅含 LLVM 的 `tvm_objs` 失败、纯 runtime 目标成功”的日志证据完全吻合。
- 显式 `-isystem /usr/include` 会被 GCC 视为对系统目录的重新指定，破坏 `#include_next` 的搜索链，于是 `<cstdlib>` 里的 `#include_next <stdlib.h>` 找不到 `/usr/include/stdlib.h`（并非该文件缺失）。将其改为 `-I /usr/include` 后，按 GCC 文档该目录与默认系统目录重复，`-I` 被忽略、目录仍按系统目录位置搜索，`#include_next` 恢复正常；LLVM 头文件仍在默认系统路径中可被找到。
- 只改这一处、不改 TVM 业务逻辑、不关闭 LLVM（保留 `USE_LLVM ON`），改动范围最小。

### 验证结果（按要求执行）
1. 已从上游获取实际源文件：从 `https://codeload.github.com/apache/tvm/tar.gz/refs/tags/v0.26.0` 下载 `apache/tvm` tag `v0.26.0`，确认 `cmake/modules/LLVM.cmake` 中带引号的 `"-isystem"` 全仓库仅出现 1 次，位于 `list(APPEND TVM_LLVM_INCLUDE_FLAGS "-isystem" "${__llvm_include_dir}")`；`re.subn`/`sed` 替换成功，结果为 `list(APPEND TVM_LLVM_INCLUDE_FLAGS "-I" "${__llvm_include_dir}")`。
2. 在真实 `openeuler/openeuler:24.03-lts-sp4` 容器内安装 `gcc-c++ llvm-devel`（GCC 12.3.1）实测：
   - `llvm-config --includedir` 返回 `/usr/include`；
   - `g++ -isystem /usr/include -c t.cc` 复现 `fatal error: stdlib.h: No such file or directory`（不可用）；
   - `g++ -I /usr/include -c t.cc` 编译通过。
3. 另用 CMake 最小工程复刻 `tvm_objs` 机制（`include_directories(SYSTEM /usr/include)` + INTERFACE 目标注入 flag）：`-isystem` 版本失败、`-I` 版本构建通过，且 `include_directories(SYSTEM ...)` 的 `/usr/include` 被 CMake 过滤，确认 patch 后不会再残留 `-isystem /usr/include`。

## 潜在风险
- 该改动把 LLVM include 目录由 `-isystem` 降级为 `-I`：对 `/usr/include` 这类默认系统目录无影响（GCC 按系统目录处理）；对外部非系统 LLVM 目录会失去系统头文件的告警抑制，属于可接受的轻微行为差异，不影响功能与构建。
- patch 依赖 v0.26.0 中 `"-isystem"` 的存在；已加 `grep -q 'TVM_LLVM_INCLUDE_FLAGS "-I"'` 断言，若上游后续版本改变该逻辑会显式构建失败而非静默产出坏镜像。
- 仅修改 Dockerfile，未改动 TVM 版本、依赖、`USE_LLVM` 及本 PR 其他文件（README/meta/image-info 未触碰），无其他关联风险。

## 验证状态说明
未在容器内完整执行 `cmake .. && cmake --build .`（TVM 全量编译耗时过长）；但已在真实 openEuler 24.03-LTS-SP4 + llvm-devel 环境下复现原始错误并验证替代 flag 可用，且最小 CMake 工程复刻链路通过。
# 修复摘要

## 修复的问题
在 bolt 新镜像的 Dockerfile 构建依赖中补充 `zlib-devel`，修复 conan 从源码构建 `cmake/3.31.10` 时其内置 `cmcurl`（`content_encoding.c`）找不到系统 `<zlib.h>` 导致的构建失败。

## 修改的文件
- `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile`: 在第 8 行的 `dnf install` 依赖清单中新增 `zlib-devel`（`libstdc++-static glibc-static libquadmath-devel` → `libstdc++-static glibc-static libquadmath-devel zlib-devel`）。

## 修复逻辑
- CI 报错为 `Utilities/cm3p/zlib.h:8:12: fatal error: zlib.h: No such file or directory`，失败于 `scripts/install-bolt-deps.sh && conan profile detect && make release` 步骤中 conan 构建 `cmake/3.31.10` 的阶段。
- 上游验证（非正则 patch，无需 `re.sub` 校验，但已核对源码）：
  - 已从上游 `https://raw.githubusercontent.com/Kitware/CMake/v3.31.10/Utilities/cm3p/zlib.h` 获取文件，确认第 8 行在 `CMAKE_USE_SYSTEM_ZLIB` 下 `#include <zlib.h>`；`Utilities/cmcurl/lib/content_encoding.c` 在 `#ifdef HAVE_LIBZ` 下 `#include <cm3p/zlib.h>`。因此编译期确实需要系统 zlib 头文件。
  - 已从上游 `https://raw.githubusercontent.com/bytedance/bolt/2d01261/scripts/install-bolt-deps.sh` 获取脚本，确认该脚本仅配置 conan remote（`conan-center-index`），**不负责安装任何系统依赖**，所以系统依赖必须在 Dockerfile 的 `dnf install` 中补齐。这印证了分析报告方向 1（高置信度）。
- openEuler 24.03-lts-sp4 基础镜像默认提供 zlib 运行库（`libz.so.*`）但不含开发头文件 `/usr/include/zlib.h`，因此安装 `zlib-devel` 即可提供该头文件，使 cmake 内置 cmcurl 编译通过，进而让 `conan_build → make release` 继续。
- 该改动仅新增一个构建期依赖，不改变任何业务/源码逻辑，也不影响镜像运行时行为。

## 潜在风险
- 本次仅补齐直接报错的 `zlib.h` 依赖。分析报告提及 cmcurl 还可能涉及 `bzip2-devel`、`libarchive-devel`、`openssl-devel` 等其他 `-devel` 包，但本次日志仅在 zlib 处失败（OpenSSL 若缺失会在 cmake configure 阶段直接 FATAL），故按最小化原则未一并添加。若后续构建在其他 `-devel` 头文件处再次失败，应在同一 `dnf install` 清单中继续补齐。
- 建议按分析报告要求，重新触发该 Dockerfile 的完整构建（x86-64 与 aarch64），确认 conan 构建 cmake 阶段不再出现 `zlib.h: No such file or directory`。
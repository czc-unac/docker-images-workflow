# 修复摘要

## 修复的问题
修复 CP2K 2026.2 镜像在 CMake 配置阶段因 DBCSR 版本号被外层 CP2K 的 git tag 污染（`2026.2.v2026.2v2026.2`）而报 `Could not find ... package "DBCSR" ... compatible with requested version "2.8"` 的构建失败。

## 修改的文件
- `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile`: 在 `git clone` 之后增加 `rm -rf /opt/cp2k/.git`，避免 DBCSR 的版本探测逻辑向上遍历找到 CP2K 仓库的 `.git`。

## 修复逻辑
### 真实 CI 证据（已获取日志，非 diff 推断）
分析报告中 `ci.logs` 缺失，因此本次修复直接从 GitCode 拉取了 Fix PR #4742 的最新失败构建日志（x86_64 job #4859、aarch64 job #4955），两个架构的失败完全一致，首个（也是唯一）错误为：

```
CMake Error at CMakeLists.txt:551 (find_package):
  Could not find a configuration file for package "DBCSR" that is compatible
  with requested version "2.8".
    /opt/cp2k/tools/toolchain/install/dbcsr-2.10.0/lib/cmake/dbcsr/DBCSRConfig.cmake, version: 2026.2.v2026.2v2026.2
```

### 根因
- CP2K 2026.2 已移除 Makefile 构建体系（上游 `support/v2026.2` 根目录无 `Makefile`），必须走 CMake。此前两次修复已分别补装 `xz`、迁移到 `build_cp2k.sh`，方向正确，故本次构建已能走到 CMake configure 阶段。
- 工具链在 `/opt/cp2k/tools/toolchain/build/dbcsr-2.10.0` 目录内编译 DBCSR 2.10.0（见 `install_dbcsr.sh`，`dbcsr_ver="2.10.0"`）。
- DBCSR 自带的 `cmake/GetGitRevisionDescription.cmake` 中 `get_git_head_revision` 会**逐级向上查找 `.git`**；由于构建目录位于 `/opt/cp2k`（由 `git clone -b support/v2026.2` 得到的 git 工作树）内部，它找到了 `/opt/cp2k/.git`，于是 `git describe` 返回 CP2K 的 tag `v2026.2`，DBCSR 将其误当作自身版本，正则解析后得到畸形版本 `2026.2.v2026.2v2026.2`，无法满足 CP2K `find_package(DBCSR 2.8)` 的最低版本要求。
- 该问题与上游 CP2K issue #5184「Incompatible version of DBCSR in toolchains」完全一致；上游给出的规避方式是不要把 DBCSR 放在其它 git 工作树内构建。

### 修复方式
在 `git clone` 完成后删除 `/opt/cp2k/.git`。这样 DBCSR 的 `get_git_head_revision` 走不到任何 `.git`，会返回以 `-NOTFOUND` 结尾的值，CMake 判定为 false，DBCSR 回退读取自带的 `VERSION` 文件（`MAJOR=2 MINOR=10 PATCH=0`），版本正确解析为 `2.10.0`，满足约束。

已验证：
- `support/v2026.2` 根目录无 `.gitmodules`（2026.2 无子模块），`--recursive` 为空操作，删除 `.git` 不影响已检出的源码。
- CP2K 的 `CMakeLists.txt` 第 29 行硬编码 `VERSION "2026.2"`，不依赖 git describe，删除 `.git` 不影响 CP2K 自身版本。
- DBCSR 构建目录 `${ROOTDIR}/build`（`ROOTDIR=/opt/cp2k/tools/toolchain`）与 `/opt/cp2k/.git` 之间不存在其它 `.git`，删除该目录即可切断向上探测。

## 潜在风险
无。仅删除构建阶段从上游克隆的临时 `.git` 目录，不改变源码内容、不改变构建参数与产物路径；对上游仓库无副作用。
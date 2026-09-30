# CI 失败分析报告

## 基本信息
- PR: #4724 — 【自动升级】fbthrift容器镜像升级至2026.09.28.00版本.
- 失败类型: `build-error`
- 置信度: 低
- 知识库匹配: 新模式
- 新模式标题: getdeps构建失败
- 新模式症状关键词: getdeps.py, exit code 1, fbthrift, libaio, _verify_hash, Dockerfile:18

## 根因分析

### 前置一致性检查
`ci.logs` 末尾为 `Build step 'Execute shell' marked build as failure` / `Finished: FAILURE`，
且日志中明确出现 Docker 构建失败（`exit code: 1`），**不属于**"日志成功但 PR 失败"的 trigger 层场景，
该失败是真实的镜像构建失败，失败阶段为 Dockerfile 第 18–23 行的 `getdeps.py ... build fbthrift` 步骤。

### 直接错误
（日志中最关键的错误信息，来自 Docker daemon）
```
#11 ERROR: process "/bin/sh -c git clone -b ${VERSION} --depth 1 https://github.com/facebook/fbthrift.git /build &&     cd /build &&     mkdir -p /tmp/fbcode_builder_getdeps-ZbuildZbuildZfbcode_builder-root/downloads &&     cp /tmp/libaio.tar.gz .../downloads/libaio-libaio-libaio-0.3.113.tar.gz &&     python3 /tmp/fix_getdeps.py &&     ./build/fbcode_builder/getdeps.py --allow-system-packages --install-prefix /usr/local build fbthrift" did not complete successfully: exit code: 1
ERROR: failed to solve: process "... build fbthrift" did not complete successfully: exit code: 1
------
Dockerfile:18
  18 | >>> RUN git clone -b ${VERSION} --depth 1 https://github.com/facebook/fbthrift.git /build && \
  19 | >>>     cd /build && \
  ...
  23 | >>>     ./build/fbcode_builder/getdeps.py --allow-system-packages --install-prefix /usr/local build fbthrift
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```
注意：`### Error lines` 段中被抽取的所谓 "error" 行（如 `boost/winapi/get_last_error.hpp`、
`.../error_codes.hpp`、`regex_error.hpp`、`cpplexer_exceptions.hpp` 等）只是**文件名包含 "error" 子串**，
并非真实错误信息；`-- Performing Test ... - Failed` / `-- Check size of ... - failed` 也是 CMake 正常的探测失败，
均不能作为根因。

### 根因定位
- 失败位置: `Others/fbthrift/2026.09.28.00/24.03-lts-sp4/Dockerfile:18-23`（`getdeps.py build fbthrift` 步骤）
- 失败原因: `getdeps.py` 构建 fbthrift 时返回退出码 1，但**日志中没有出现导致失败的第一条真实错误**
  （无 Python traceback、无编译器错误、无 CMake Error、无下载/校验报错），提供的 `ci.logs` 是被截断且
  stdout/stderr 交错刷新的片段，无法据此确定失败的具体依赖或编译单元。

### 与 PR 变更的关联
强关联。失败步骤是本 PR 新增的 `Others/fbthrift/2026.09.28.00/24.03-lts-sp4/Dockerfile` 中的
`getdeps.py build fbthrift`，同级新增了 `fix_getdeps.py`（对上游 `getdeps_platform.py`、`fetcher.py`、
`libaio` manifest 做字符/正则替换）。本次自动升级把 `VERSION` 提升到 `v2026.09.28.00`，
上游 getdeps 脚本、manifest 与依赖版本可能相对 2026.09.21.00 发生变化，因此该改动直接触发了失败。

### 日志能确认的进度（用于排除）
- 平台识别与依赖构建大部分已通过：`CMAKE_PREFIX_PATH` 中已列出
  `libaio-...`, `libdwarf-...`, `libevent-...` 等安装目录，日志中可见
  `boost`、`googletest`、`libdwarf`、`libevent`、`lz4` 的 configure/build/install 完成。
- 说明 `fix_getdeps.py` 中 libaio 相关的预置 tarball 与 `_verify_hash` 处理**至少没有在早期阶段阻断构建**，
  失败发生在后续依赖或 fbthrift 自身构建阶段（日志尾部为 `Building lz4...` 后即报错，但输出顺序不可靠）。

## 修复方向

### 方向 1（置信度: 低）
先获取未截断的完整 `getdeps.py` 构建日志，定位第一条真实错误（Python traceback / `make`、`cmake` 报错 /
下载或 hash 校验错误），再针对该具体依赖或编译单元处理。当前日志不足以支撑更精确的修复方向。

### 方向 2（置信度: 低）
核查本 PR 新增的 `fix_getdeps.py` 三个替换在 fbthrift `v2026.09.28.00` 上游文件中是否仍然生效且语义正确：
1. `getdeps_platform.py` 中发行版元组字符串是否与上游一致（字符串不匹配则 `openeuler` 识别失效）；
2. `fetcher.py` 中 `_verify_hash` 的正则是否能匹配上游该版本的实际方法（签名、是否位于文件末尾其后不再有 `    def `），
   以及替换后方法签名是否与调用方一致；
3. `libaio` manifest 的 `subdir` 是否与预置 tarball 内部顶层目录名一致。
三者任一生效失效都可能导致 `getdeps.py` 以退出码 1 失败。

### 方向 3（置信度: 低）
对比 2026.09.21.00 → 2026.09.28.00 上游 getdeps 依赖清单（folly/fizz/wangle/libevent/fmt/boost 等）的版本变化，
排查新版本依赖在 openEuler 24.03-LTS-SP4 工具链下的编译兼容性问题。

## 需要进一步确认的点
1. **（最关键）未截断的完整构建日志**：需要 Dockerfile:18-23 这一步的完整输出，尤其是失败瞬间的
   Python traceback 或 `make`/`cmake`/`ninja` 的第一条错误行。当前 `ci.logs` 仅含进度信息，未含真实错误，属**证据不足**。
2. fbthrift `v2026.09.28.00` 上游 `build/fbcode_builder/getdeps/fetcher.py` 中 `_verify_hash` 的实际签名与所在位置。
3. 同版本 `build/fbcode_builder/getdeps/getdeps_platform.py` 中发行版元组的确切内容。
4. 同版本 `build/fbcode_builder/manifests/libaio` 的 `subdir` / `sha256` 定义，以及预置 `libaio-libaio-0.3.113.tar.gz` 解包后的顶层目录名。
5. 2026.09.28.00 相对 2026.09.21.00 的依赖版本差异清单。
6. 由于提供的日志被截断且交错，修复前**不应假设**上述任一方向即为根因。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
本项目 `fix_getdeps.py` 明确包含"用正则/字符串替换第三方 getdeps 源文件内容"的做法，因此 code-fixer 在提交前必须：
- 依据 Dockerfile 中 `ARG VERSION=v2026.09.28.00`，从 facebook/fbthrift 拉取该 tag 下的
  `build/fbcode_builder/getdeps/fetcher.py`，确认 `_verify_hash` 方法在 2026.09.28.00 的**实际签名**
  （参数个数与顺序）及其是否还有后续 `    def ` 方法，验证 `fix_getdeps.py` 中的正则确实能匹配并只替换该方法；
- 同样拉取 `build/fbcode_builder/getdeps/getdeps_platform.py`，逐字核对被 `str.replace` 的发行版元组字符串是否存在；
- 拉取 `build/fbcode_builder/manifests/libaio`，核对 `subdir` 与预置 tarball 内部目录名是否一致；
- 在确认上述匹配关系后再提交，且**必须先取得完整失败日志**以确认根因，不得在证据不足时直接套用上述任一修复方向。

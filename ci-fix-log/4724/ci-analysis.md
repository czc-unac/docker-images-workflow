# CI 失败分析报告

## 基本信息
- PR: #4724 — 【自动升级】fbthrift容器镜像升级至2026.09.28.00版本.
- 失败类型: build-error
- 置信度: 低
- 知识库匹配: 新模式
- 新模式标题: getdeps依赖构建失败
- 新模式症状关键词: getdeps, fbcode_builder, libaio, did not complete successfully, exit code: 1

> 前置检查结论：`ci.logs` 末尾为 `Finished: FAILURE`，且 Docker 明确报 `failed to solve ... exit code: 1`，
> 不属于"日志显示 SUCCESS 但 PR 失败"的 infra-error 场景。失败发生在真实构建阶段，但触发非零退出的
> 具体子命令错误在所提供的日志中**并未出现**（见下文"证据不足"）。

## 根因分析

### 直接错误
```
#11 ERROR: process "/bin/sh -c git clone -b ${VERSION} --depth 1 https://github.com/facebook/fbthrift.git /build &&     cd /build &&     mkdir -p /tmp/fbcode_builder_getdeps-ZbuildZbuildZfbcode_builder-root/downloads &&     cp /tmp/libaio.tar.gz /tmp/fbcode_builder_getdeps-ZbuildZbuildZfbcode_builder-root/downloads/libaio-libaio-libaio-0.3.113.tar.gz &&     python3 /tmp/fix_getdeps.py &&     ./build/fbcode_builder/getdeps.py --allow-system-packages --install-prefix /usr/local build fbthrift" did not complete successfully: exit code: 1
...
Dockerfile:18
  18 | >>> RUN git clone -b ${VERSION} --depth 1 https://github.com/facebook/fbthrift.git /build && \
  19 | >>>     cd /build && \
  20 | >>>     mkdir -p /tmp/.../downloads && \
  21 | >>>     cp /tmp/libaio.tar.gz /tmp/.../downloads/libaio-libaio-libaio-0.3.113.tar.gz && \
  22 | >>>     python3 /tmp/fix_getdeps.py && \
  23 | >>>     ./build/fbcode_builder/getdeps.py --allow-system-packages --install-prefix /usr/local build fbthrift
ERROR: failed to solve: ... did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Others/fbthrift/2026.09.28.00/24.03-lts-sp3/Dockerfile:18-23`（`git clone && fix_getdeps.py && getdeps.py ... build fbthrift` 这一步），进一步定位到 `fix_getdeps.py` 所修补的 `build/fbcode_builder/getdeps/` 构建流程内部。
- 失败原因: **证据不足，无法确定具体子命令**。所提供的日志（grep 结果 + Build tail）在时间戳 `308.6s` 处中断于 `Building lz4...`，随后直接是 Docker 的整条 RUN 非零退出，缺失真正报错的那段输出。

### 与 PR 变更的关联
本 PR 为自动升级，新增了 `2026.09.28.00/24.03-lts-sp4/` 目录，其中：
- `Dockerfile`：`ARG VERSION=v2026.09.28.00`，在 `getdeps.py ... build fbthrift` 前执行 `fix_getdeps.py`，并预置 `libaio-libaio-0.3.113.tar.gz`。
- `fix_getdeps.py`：对上游 `build/fbcode_builder/getdeps/` 下的**外部源文件**做三处正则/字符串修补：
  1. `getdeps_platform.py` 的发行版元组加入 `"openeuler"`；
  2. 用正则将 `fetcher.py` 的 `_verify_hash` 方法整体替换为 `pass`（跳过 libaio 哈希校验）；
  3. `manifests/libaio` 的 `subdir = libaio-libaio-0.3.113` 改为 `subdir = libaio-0.3.113`。

失败点正是这一步，因此**与本次 PR 改动直接相关**：新版本的上游 `getdeps` 脚本内容（方法签名、缩进、字段名）一旦与正则/字符串不匹配，修补会静默失败或破坏文件，进而导致 `getdeps.py build` 非零退出。但当前日志未给出足以判定是哪一处（及是否匹配）报错的信息。

### 关键观察（不构成根因，仅供定位）
- 日志中大量 `Performing Test ... - Failed` / `Check size of SSIZE_T - failed` 均为 CMake **特性探测的预期否定结果**，属非致命信息，**不能**作为根因。
- `CMAKE_PREFIX_PATH` 中已出现 `libaio-...` 前缀，暗示 libaio 可能已处理，但日志中并无明确的 `Building libaio...` 成功行，无法据此确认 `_verify_hash` 跳过是否生效。
- 预置文件名 `libaio-libaio-libaio-0.3.113.tar.gz`（三个 `libaio`）与常规 GitHub 归档命名不一致，是否与 getdeps 期望的缓存文件名匹配，需在上游 fetcher 代码中核对。

## 修复方向

### 方向 1（置信度: 低）
获取并定位 `getdeps.py` 输出中第一处真正的 error（本日志已截断，不在文件内），据其确定是哪个依赖/哪一步失败，再针对性修补。
这是优先级最高的方向——在拿到真实报错前，任何具体修复都属于猜测。

### 方向 2（置信度: 低）
若确认失败源于 `fix_getdeps.py` 对上游 `build/fbcode_builder/getdeps/` 的修补未命中或误伤，则需针对 `v2026.09.28.00` 的实际上游源码重新校准三处修补（发行版元组、`_verify_hash` 签名/缩进、libaio manifest 字段），使修补真正生效且不破坏文件。

## 需要进一步确认的点
1. **完整的下游 getdeps 构建日志**：定位 `Building <某依赖>...` 之后、`exit code: 1` 之前的第一条真实 error（Docker 构建号 `#11` 的完整输出）。当前只有 308.6s 处的尾部片段。
2. **上游 `fetcher.py` 中 `_verify_hash` 的真实签名与后续方法**：确认 `fix_getdeps.py` 的正则 `def _verify_hash\(self[^)]*\)[^:]*:.*?(?=\n    def )` 在 `v2026.09.28.00` 版本上能否命中；特别是方法是否为类内最后一个方法（缺少后续 `    def ` 将导致 lookahead 永不匹配、替换静默失效）。
3. **上游 `getdeps_platform.py` 发行版元组**：确认 `("fedora", "centos", "centos_stream", "rocky", "alma")` 字样是否仍存在（否则字符串替换静默无效）。
4. **上游 `manifests/libaio` 的 `subdir` 值**：确认原值确为 `libaio-libaio-0.3.113`，以及预置 tarball 内部真实目录名。
5. **预置下载文件名一致性**：确认 getdeps 期望的缓存文件名，核对 `libaio-libaio-libaio-0.3.113.tar.gz` 是否会被识别，避免因文件名不匹配触发重新下载/校验。
6. **各依赖是否真的构建成功**：日志中 `libaio`、`libevent`、`lz4` 等是否都已完成安装，以排除/锁定失败依赖。

## 修复验证要求
本次修复方向涉及"修改正则/字符串以匹配第三方（上游 fbthrift）源文件中的内容"，因此 code-fixer 在提交前**必须**执行以下验证，不得假设修补一定命中：

1. code-fixer 必须从上游仓库按 Dockerfile 中的 `ARG VERSION=v2026.09.28.00`（即 fbthrift tag `v2026.09.28.00`）拉取对应文件，逐项验证修补确实命中：
   - `build/fbcode_builder/getdeps/fetcher.py` 中 `_verify_hash` 方法的**实际签名与缩进**，确认正则 `def _verify_hash\(self[^)]*\)[^:]*:.*?(?=\n    def )` 能匹配；
   - `build/fbcode_builder/getdeps/getdeps_platform.py` 中发行版元组字符串是否仍为 `("fedora", "centos", "centos_stream", "rocky", "alma")`；
   - `build/fbcode_builder/manifests/libaio` 中 `subdir` 字段的实际取值。
2. 必须在本机对 `v2026.09.28.00` 的 `getdeps` 源码实际运行 `fix_getdeps.py`，确认三处修补均产生预期差异且未破坏文件（尤其正则替换后 `fetcher.py` 仍可被 Python 正常解析/导入）。
3. 在获得完整 `getdeps` 构建错误日志并确认根因前，不得将本文档"方向 2"当作已验证结论；若无法复现真实报错，应回到"方向 1"，先取得缺失的错误日志再行修复。

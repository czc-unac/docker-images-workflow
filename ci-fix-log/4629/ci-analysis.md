# CI 失败分析报告

## 基本信息
- PR: #4629 — 【自动升级】fbthrift容器镜像升级至2026.09.21.00版本.
- 失败类型: build-error（证据不足）
- 置信度: 低
- 知识库匹配: 新模式
- 新模式标题: getdeps构建失败
- 新模式症状关键词: getdeps.py, build fbthrift, exit code: 1, fbcode_builder, libaio, _verify_hash

## 根因分析

### 直接错误
```
#11 301.2 Building lz4...
#11 ERROR: process "/bin/sh -c git clone -b ${VERSION} --depth 1 https://github.com/facebook/fbthrift.git /build &&     cd /build &&     mkdir -p /tmp/fbcode_builder_getdeps-ZbuildZbuildZfbcode_builder-root/downloads &&     cp /tmp/libaio.tar.gz /tmp/fbcode_builder_getdeps-ZbuildZbuildZfbcode_builder-root/downloads/libaio-libaio-libaio-0.3.113.tar.gz &&     python3 /tmp/fix_getdeps.py &&     ./build/fbcode_builder/getdeps.py --allow-system-packages --install-prefix /usr/local build fbthrift" did not complete successfully: exit code: 1
ERROR: failed to solve: process "/bin/sh -c git clone -b ${VERSION} ..." did not complete successfully: exit code: 1
------
Dockerfile:18
  18 | >>> RUN git clone -b ${VERSION} --depth 1 https://github.com/facebook/fbthrift.git /build && \
  ...
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `Others/fbthrift/2026.09.21.00/24.03-lts-sp4/Dockerfile:18`（`getdeps.py ... build fbthrift` 步骤，即新增文件）
- 失败原因: 无法确定。日志末尾只给出 Docker 构建进程 `exit code: 1`，**未包含真正触发退出的子包错误输出**（既没有 C++ 编译错误，也没有下载/哈希校验失败的堆栈）。日志中大量 `-- Performing Test ... - Failed`、`Check size of ... failed` 属于 CMake 特性探测的正常噪声，`-- Installing: .../boost/...` 为正常安装输出，均非根因。

### 日志可见的构建进度
- 已成功走过 boost、googletest、libdwarf、libevent、lz4 等依赖（CMAKE_PREFIX_PATH 中已出现 `libaio`、`libdwarf`、`libevent` 的安装前缀），说明 `fix_getdeps.py` 中的 openEuler 平台识别与 libaio subdir 修正在前期至少未立即崩溃。
- 失败发生在 `Building lz4...` 之后的下一个依赖/阶段，但该阶段的具体报错行缺失（日志被截断，`### Build tail` 只保留末尾片段）。

### 与 PR 变更的关联
PR 新增了该目录的 Dockerfile 与 `fix_getdeps.py`，CI 失败正好落在新增文件对应的构建步骤内，因此与 PR 改动高度相关；但**是 `fix_getdeps.py` 的哪一条 patch（平台识别 / `_verify_hash` 正则 / libaio subdir）导致，还是上游 v2026.09.21.00 源码变化导致后面某个依赖构建失败，从现有日志无法区分**。

## 修复方向

### 方向 1（置信度: 低）
先获取该 job 未截断的完整日志（失败子包的实际 stderr），定位退出前最后一个失败的依赖/命令，再决定修复对象。当前日志不足以指向具体修复。

### 方向 2（置信度: 低）
若无法拿到完整日志，优先核查 `fix_getdeps.py` 第 2 步通过正则替换 `_verify_hash` 的写法在 fbthrift v2026.09.21.00 的 `build/fbcode_builder/getdeps/fetcher.py` 中是否真的匹配成功：
- 若正则与上游实际方法签名/缩进不符，`re.sub` 会静默不替换，哈希校验仍然生效，后续某个下载依赖（非预置的 libaio）可能因哈希/来源变化在评估阶段失败。
- 同时核对 Dockerfile 预置 tarball 的名称 `libaio-libaio-libaio-0.3.113.tar.gz` 与 `libaio` manifest 期望的下载文件名/`subdir` 是否一致，避免预置文件被忽略而触发上游下载。

## 需要进一步确认的点
1. 获取完整（未截断）的 CI 构建日志，确认 `exit code: 1` 前最后一个失败的 getdeps 子命令与报错原文。
2. 对照 fbthrift v2026.09.21.00 上游 `build/fbcode_builder/getdeps/fetcher.py`，确认 `_verify_hash` 方法的真实签名/缩进，验证 `fix_getdeps.py` 的正则能否匹配。
3. 对照上游 `build/fbcode_builder/manifests/libaio`，确认 `subdir`、下载文件名与预置 tarball 内部目录结构、Dockerfile 中 `cp` 目标名三者是否一致。
4. 确认 getdeps 在 lz4 之后下一个待构建依赖（lzma/snappy/libsodium 等）以及 fbthrift 自身是否出现编译/链接错误。

## 修复验证要求
本 PR 的 `fix_getdeps.py` 属于"修改正则以匹配第三方/上游源文件中的内容"，且修复方向（方向 2）也依赖该正则。
若 Code Fixer 据此调整正则，必须从上游 fbthrift `${VERSION}`（即 v2026.09.21.00）的
`build/fbcode_builder/getdeps/fetcher.py` 拉取实际源码，确认 `_verify_hash` 方法的真实签名后，验证新正则确实能匹配目标内容再提交。
若最终无法取得完整失败日志，应判定为证据不足，不应在无日志依据的情况下盲改 patch。

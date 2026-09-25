# CI 失败分析报告

## 基本信息
- PR: #4464 — 【自动升级】fbthrift容器镜像升级至2026.09.21.00版本.
- 失败类型: build-error
- 置信度: 低
- 知识库匹配: 模式42（日志缺失无法定位）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#11 835.3 Building lz4...
#11 ERROR: process "/bin/sh -c git clone -b ${VERSION} --depth 1 https://github.com/facebook/fbthrift.git /build &&     cd /build &&     mkdir -p /tmp/fbcode_builder_getdeps-ZbuildZbuildZfbcode_builder-root/downloads &&     cp /tmp/libaio.tar.gz /tmp/fbcode_builder_getdeps-ZbuildZbuildZfbcode_builder-root/downloads/libaio-libaio-libaio-0.3.113.tar.gz &&     python3 /tmp/fix_getdeps.py &&     ./build/fbcode_builder/getdeps.py --allow-system-packages --install-prefix /usr/local build fbthrift" did not complete successfully: exit code: 1
ERROR: failed to solve: process "/bin/sh -c git clone -b ${VERSION} ..." did not complete successfully: exit code: 1
Dockerfile:18
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `Others/fbthrift/2026.09.21.00/24.03-lts-sp4/Dockerfile:18-23`（第 5 步 `getdeps.py build fbthrift`），**无法进一步定位到具体依赖或源文件**
- 失败原因: **证据不足**。提供的日志中不存在任何真正的编译/工具错误文本，只有 Docker 层返回的通用 `exit code: 1`，失败子步骤的具体输出被截断。

### 日志证据分析（为什么判定为证据不足）
- 日志头部 "Error lines (newest first)" 中的所谓错误行均为**误报**：
  - boost/libdwarf 的 `-- Installing: .../error_codes.hpp`、`dwarf_error.c.o`、`cpp_value_error.hpp` 等，是**文件名里包含 "error"** 而被关键词抓取，并非错误。
  - `-- Performing Test HAVE_NO_UNNAMED_TYPE_TEMPLATE_ARGS - Failed`、`-- Check size of SSIZE_T - failed`、`EVENT__HAVE_DECL_CTL_KERN - Failed` 等是 CMake 正常的**特性探测失败**，属预期行为。
- 日志中不存在 `error:`（编译器报错）、`Traceback`/`Exception`（getdeps 的 Python 异常）、`hash`/`sha256` 校验失败、`No such file`、`fatal:` 等可定位根因的关键字。
- "Build tail" 的时序明显错乱（libevent、lz4 的"Assessing/Building/下载/解压"重复出现），末尾停在 `Building lz4...`，不能据此断定失败发生在 lz4；其后的真实错误输出缺失。
- 日志以 `Finished: FAILURE` 结束，说明确实是构建失败（不属于"日志成功但 PR 失败"的场景），但**失败原因未被日志捕获**。

### 与 PR 变更的关联
本 PR 为自动升级新增 `Others/fbthrift/2026.09.21.00/24.03-lts-sp4/` 全量构建文件（Dockerfile、`fix_getdeps.py`、预置 `libaio-libaio-0.3.113.tar.gz`、README/meta/image-info 条目），并注册到 `meta.yml`。构建失败发生在该新增 Dockerfile 的 getdeps 构建步骤，故与本次 PR 直接相关，但**具体是哪个依赖/哪一行导致失败无法从现有日志确认**。

## 修复方向

### 方向 1（置信度: 低）
获取该次 docker build 的**完整 getdeps 构建日志**（尤其是 `exit code: 1` 之前 getdeps 抛出的 Python 异常栈或失败依赖的 cmake/ninja 输出），据此定位到具体失败的依赖（如 folly / fbthrift 本体或某个第三方库）后再修复。当前信息不足以给出任何具体修复动作。

### 方向 2（置信度: 低）
检查本次新增的 `fix_getdeps.py` 中针对上游 `build/fbcode_builder/getdeps/fetcher.py` 的 `_verify_hash` 方法所做的正则替换是否真正生效：若上游 v2026.09.21.00 的方法签名/缩进/后续方法名与正则不匹配，`re.sub` 会静默返回原文，导致 patch 未生效。但不能据此认定本次失败根因——日志中**没有**哈希校验失败的证据，且 libaio 已出现在 `CMAKE_PREFIX_PATH`（`/usr/local/libaio-5IyzYOju...`），说明 libaio 已成功安装，故该假设需先以完整日志验证。

### 方向 3（置信度: 低）
对比上一可用版本 `2026.09.14.00` 的构建文件，确认 `fix_getdeps.py` 依赖的上游代码结构（`getdeps_platform.py` 的发行版元组、`manifests/libaio` 的 `subdir` 字段）在 2026.09.21.00 是否发生变化，导致其中的 `str.replace` 静默失配。同样需要完整日志佐证。

## 需要进一步确认的点
1. 触发失败的**下游/详细构建 job 完整日志**：`EXIT code: 1` 前 getdeps 打印的异常与失败的依赖名/命令（当前提供的日志被截断，无法定位）。
2. `fetch_getdeps` 相关补丁是否生效：需拉取 fbthrift `v2026.09.21.00` 的 `build/fbcode_builder/getdeps/fetcher.py`，确认 `_verify_hash` 的实际签名与后续方法名，验证 `fix_getdeps.py` 中正则能否匹配。
3. 该版本 `build/fbcode_builder/getdeps/getdeps_platform.py` 中发行版元组 `("fedora", "centos", "centos_stream", "rocky", "alma")` 是否仍然存在。
4. 该版本 `build/fbcode_builder/manifests/libaio` 原始 `subdir` 字段值是否为 `libaio-libaio-0.3.113`（与预置 tarball 内部目录结构是否一致）。
5. 日志中 `Building lz4...` 之后缺失的输出内容，确认失败是否发生在 lz4 之后的某个依赖。

## 修复验证要求
> 说明：本报告置信度为"低"，以下验证为后续修复的前置条件，code-fixer **不得假设** `fix_getdeps.py` 的正则一定正确。

1. **必须先取得完整构建日志**，确认失败的具体依赖与命令；在无完整日志前，禁止仅凭本报告的推测方向提交修复。
2. 若修复方向涉及"修改正则以匹配第三方/上游源文件"，code-fixer 必须从上游仓库按 Dockerfile 的 `ARG VERSION=v2026.09.21.00` 拉取对应文件并验证正则匹配：
   - 从 `https://github.com/facebook/fbthrift.git` 的 tag `v2026.09.21.00` 获取 `build/fbcode_builder/getdeps/fetcher.py`，确认 `_verify_hash` 方法**实际的完整签名、缩进及紧随其后的方法定义**，验证 `def _verify_hash\(self[^)]*\)[^:]*:...(?=\n    def )` 能命中目标文本。
   - 同样核对该版本的 `getdeps_platform.py` 发行版元组与 `manifests/libaio` 的 `subdir` 字段实际内容，验证两处 `str.replace` 的源字符串存在。
3. 验证方式建议：在本地对上游对应版本文件执行补丁脚本后，diff 确认目标位置已被实际修改（而非 `re.sub` 静默未匹配），再提交。

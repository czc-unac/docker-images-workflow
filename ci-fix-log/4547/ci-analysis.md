# CI 失败分析报告

## 基本信息
- PR: #4547 — 【自动升级】fbthrift容器镜像升级至2026.09.21.00版本.
- 失败类型: build-error
- 置信度: 低
- 知识库匹配: 模式42（日志缺失无法定位）
- 新模式标题: （不适用，已匹配现有模式）
- 新模式症状关键词: （不适用）

> 说明：本次日志**以 `Finished: FAILURE` 结尾**，不属于"日志显示成功但 PR 失败"的情形，因此执行正常分析流程。但提供的 `ci.logs` 在真正报错处被**截断**——`Build tail` 停在 `Building lz4...` 后直接跟 `exit code: 1`，缺少失败点的编译/脚本错误输出。因此无法定位具体根因，结论为"证据不足"。

## 根因分析

### 直接错误
```
#11 720.3 Building lz4...
#11 ERROR: process "/bin/sh -c git clone -b ${VERSION} --depth 1 https://github.com/facebook/fbthrift.git /build && ... && python3 /tmp/fix_getdeps.py && ./build/fbcode_builder/getdeps.py --allow-system-packages --install-prefix /usr/local build fbthrift" did not complete successfully: exit code: 1
ERROR: failed to solve: process "..." did not complete successfully: exit code: 1
------
Dockerfile:18
```

`### Error lines` 中绝大多数条目（`Installing .../error_codes.hpp`、`-- Performing Test ... - Failed`、`-- Check size of SSIZE_T - failed`）是文件名含 "error" 或 CMake 特性探测的正常输出，**并非错误**；唯一的真实错误就是上述 `exit code: 1`，但其前因被截断。

### 根因定位
- 失败位置: `Others/fbthrift/2026.09.21.00/24.03-lts-sp4/Dockerfile:18`（`RUN git clone ... && python3 /tmp/fix_getdeps.py && ./build/fbcode_builder/getdeps.py ... build fbthrift`）
- 失败原因: getdeps.py 在构建 fbthrift 依赖链的过程中以 exit code 1 失败，失败时正处于构建 `lz4` 之后的阶段；具体触发错误的子步骤与错误文本未包含在提供的日志中，无法确定。

### 与 PR 变更的关联
- 本 PR 为自动升级，新增了 `2026.09.21.00/24.03-lts-sp4/Dockerfile` 与 `fix_getdeps.py`（正则 patch 上游 fbthrift 源码），并登记 README/meta.yml/image-info.yml。
- 失败发生在**该新增 Dockerfile 的 getdeps 构建步骤**中，与本次改动直接相关（新文件）。
- 但无法从日志判断根因是 `fix_getdeps.py` 的正则/字符串替换失败，还是上游某个第三方依赖（lz4 之后）或 fbthrift 自身的编译错误。
- 从日志可见的旁证：`libaio` 已成功构建（已出现在 `CMAKE_PREFIX_PATH`/`LD_LIBRARY_PATH` 中），`boost`、`googletest`、`libdwarf`、`libevent` 均已完成，故 `fix_getdeps.py` 中 libaio 相关的两处（`_verify_hash` 跳过、manifest subdir 修正）在本版本大概率已生效。

## 修复方向

### 方向 1（置信度: 低）
先补齐证据：获取失败 job 完整日志中 `Building lz4...` 之后的内容（包括 getdeps 打印的实际错误、失败依赖名与编译器/脚本输出），据此再做定位。**在拿到该段日志前不应做任何代码改动。**

### 方向 2（置信度: 低）
若补齐日志后确认失败与 `fix_getdeps.py` 的正则/替换相关（例如 hash 校验、下载 URL、manifest 解析报错），则需对照 fbthrift `v2026.09.21.00` 的上游实际文件内容，重新校正替换规则。注意：`fetcher.py` 的 `_verify_hash` 签名字符串、`getdeps_platform.py` 的发行版元组、`manifests/libaio` 的 `subdir` 行均可能随上游版本变化，而 `re.sub`/`str.replace` 未命中时会**静默不生效**，导致后续构建在别处才报错。

### 方向 3（置信度: 低）
若补齐日志后确认为某第三方依赖在 lz4 之后构建失败（编译错误/配置错误），则属依赖链问题，需针对该依赖单独定位，与 `fix_getdeps.py` 无关。

## 需要进一步确认的点
1. 获取完整 CI 日志中 `Building lz4...` 之后的段落，确定：
   - 是哪个依赖/步骤失败；
   - 失败时打印的实际错误文本（编译器报错、Python 异常、下载/哈希错误等）。
2. 确认失败阶段：仍处于 getdeps 依赖构建，还是已进入 fbthrift 自身编译。
3. 逐项核对 `fix_getdeps.py` 对 fbthrift `v2026.09.21.00` 上游文件的命中情况：
   - `build/fbcode_builder/getdeps/getdeps_platform.py` 中的发行版元组 `("fedora", "centos", "centos_stream", "rocky", "alma")`；
   - `build/fbcode_builder/getdeps/fetcher.py` 中 `_verify_hash` 的实际签名（正则 `def _verify_hash\(self[^)]*\)[^:]*:.*?(?=\n    def )` 是否匹配）；
   - `build/fbcode_builder/manifests/libaio` 中 `subdir = libaio-libaio-0.3.113` 字符串。
4. 注意日志时间戳显示构建已进行约 720s（12 分钟），排除是否为构建超时（当前证据不支持也不排除）。

## 修复验证要求
本 PR 的修复涉及对第三方/上游源文件进行正则/字符串 patch，因此：

- code-fixer 在提交任何修改前，**必须从上游仓库 `facebook/fbthrift` 的 `v2026.09.21.00` tag**（以 Dockerfile 中 `ARG VERSION=v2026.09.21.00` 为准）拉取以下文件并验证替换规则确实命中：
  - `build/fbcode_builder/getdeps/fetcher.py` 的 `_verify_hash` 方法实际签名；
  - `build/fbcode_builder/getdeps/getdeps_platform.py` 的发行版元组原文；
  - `build/fbcode_builder/manifests/libaio` 的 `subdir` 行原文。
- 由于 `re.sub`/`str.replace` 未命中时静默不报错，必须验证替换**前后内容确实发生变化**，不能仅凭脚本退出码为 0 判定成功。
- **在补齐方向 1 所需的 `Building lz4...` 之后日志前，不应假设根因一定在 `fix_getdeps.py`**；需先确认失败的具体依赖与错误类型，再决定是否修改替换规则。

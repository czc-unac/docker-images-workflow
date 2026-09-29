# CI 失败分析报告

## 基本信息
- PR: #4724 — 【自动升级】fbthrift容器镜像升级至2026.09.28.00版本.
- 失败类型: build-error
- 置信度: 低
- 知识库匹配: 新模式
- 新模式标题: getdeps构建中断
- 新模式症状关键词: exit code: 1, getdeps.py, build fbthrift, Building lz4, 日志截断

## 根因分析

### 直接错误
```
#11 ERROR: process "/bin/sh -c git clone -b ${VERSION} --depth 1 https://github.com/facebook/fbthrift.git /build &&     cd /build &&     mkdir -p /tmp/fbcode_builder_getdeps-ZbuildZbuildZfbcode_builder-root/downloads &&     cp /tmp/libaio.tar.gz /tmp/fbcode_builder_getdeps-ZbuildZbuildZfbcode_builder-root/downloads/libaio-libaio-libaio-0.3.113.tar.gz &&     python3 /tmp/fix_getdeps.py &&     ./build/fbcode_builder/getdeps.py --allow-system-packages --install-prefix /usr/local build fbthrift" did not complete successfully: exit code: 1
...
Dockerfile:18
  18 | >>> RUN git clone -b ${VERSION} --depth 1 https://github.com/facebook/fbthrift.git /build && \
```
日志「Build tail」中 `getdeps.py` 的最后一条可见输出为：
```
#11 308.6 Extract .../lz4-lz4-1.10.0.tar.gz -> ...
#11 308.6 Building lz4...
#11 ERROR: process "..." did not complete successfully: exit code: 1
```
即：**在 `Building lz4...` 之后没有任何 CMake Error、编译 error、Python traceback 或下载报错，进程直接以 exit code 1 结束**。

### 根因定位
- 失败位置: `Others/fbthrift/2026.09.28.00/24.03-lts-sp4/Dockerfile:18`（`getdeps.py ... build fbthrift` 步骤）
- 失败原因: **证据不足**。提供的 `ci.logs` 中未包含导致 exit code 1 的底层错误信息，日志在 lz4 构建处被截断/丢失，无法确定是哪个依赖或编译单元、以及何种错误（编译错误 / hash 校验失败 / 构建脚本异常）导致失败。

### 与 PR 变更的关联
- PR 为 fbthrift 自动升级，新增 `2026.09.28.00/24.03-lts-sp4/` 目录，并**首次引入 `fix_getdeps.py`**（PR diff 中为新文件）：
  1. 在 `getdeps_platform.py` 中把 `openeuler` 加入 fedora/centos 系发行版识别列表；
  2. 通过正则把 `fetcher.py` 的 `_verify_hash` 方法整体替换为空实现（跳过 libaio 哈希校验）；
  3. 把 `manifests/libaio` 的 `subdir` 由 `libaio-libaio-0.3.113` 改为 `libaio-0.3.113`。
- 日志显示 `libaio` 已成功构建（路径 `/usr/local/libaio-5IyzYOju1uYPdEYjndKgmKnKRKzFbYACdV3tp7sYRdU` 出现），说明前 3 个 patch 至少未立即报错、构建已深入到后续依赖；因此失败**并非发生在这段 patch 脚本本身**，而是发生在后续 getdeps 构建过程中。
- 由于缺少实际报错行，无法判定该失败是否由本次 `fix_getdeps.py` 的 patch 未生效（如上游 `fetcher.py` / `manifests/libaio` 结构在新版本中变化，导致正则或字符串替换静默失配）间接引起，属于**待验证假设，不能据此下结论**。

## 修复方向

### 方向 1（置信度: 低）
需先取得完整构建日志，定位 `Building lz4...` 之后到 exit code 1 之间的真实报错。若报错为某个依赖的配置/编译失败，按知识库模式10（缺 -devel 构建依赖）或模式15（编译器错误）方向处理；若报错为 hash 校验失败，则需检查 `fix_getdeps.py` 第 2、3 个 patch 是否在 `v2026.09.28.00` 源码上真正生效。

### 方向 2（置信度: 低）
检查 `fix_getdeps.py` 中基于正则/字符串替换的 patch 的健壮性：
- `_verify_hash` 的替换正则 `r'def _verify_hash\(self[^)]*\)[^:]*:.*?(?=\n    def )'` 依赖「紧随其后存在 4 空格缩进的 `def `」，若上游方法顺序/定义形式改变，`re.sub` 会**静默不替换**（不报错），导致哈希校验仍生效；
- `subdir = libaio-libaio-0.3.113` 的字符串替换在内容变化时同样静默失效。
这两处静默失败可能使后续构建行为与预期不符，需结合完整日志判断。

## 需要进一步确认的点
1. **获取 `#11` 步骤的完整日志**（尤其是从 `Building lz4...` 到 exit code 1 之间的全部 stdout/stderr），当前提供日志在该处被截断，是根因无法确定的首要原因。
2. 确认该失败发生在 x86-64 还是 aarch64 架构 job（当前日志显示 `CMAKE_SYSTEM_PROCESSOR: x86_64`），另一架构 job 是否同样失败。
3. 从上游 fbthrift `v2026.09.28.00` 拉取 `build/fbcode_builder/getdeps/fetcher.py` 与 `build/fbcode_builder/manifests/libaio`，确认：
   - `_verify_hash` 是否仍以 `def _verify_hash(self, ...)` + 4 空格缩进 `def` 续接的形式存在；
   - `manifests/libaio` 中 `subdir` 原值是否为 `libaio-libaio-0.3.113`。
4. 确认失败是否为 Folly/fizz/wangle/mvfst 等 C++ 依赖的编译错误（日志中尚未出现相关编译输出）。

## 修复验证要求
- 本次修复若涉及调整 `fix_getdeps.py` 中针对第三方/上游源文件的正则或字符串替换（`getdeps_platform.py`、`fetcher.py` 的 `_verify_hash`、`manifests/libaio`），**code-fixer 必须先按 Dockerfile 中 `ARG VERSION=v2026.09.28.00` 从 `facebook/fbthrift` 拉取该版本对应文件**，核对目标方法的真实签名与上下文，确认替换正则/字符串确实能匹配后再提交，**不得假设替换一定生效**。
- 在未取得 `Building lz4...` 之后的实际报错前，**不应将任何单一路径当作确定根因**；优先要求补齐完整日志后再实施针对性修复。

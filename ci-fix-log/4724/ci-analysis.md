# CI 失败分析报告

## 基本信息
- PR: #4724 — 【自动升级】fbthrift容器镜像升级至2026.09.28.00版本.
- 失败类型: build-error
- 置信度: 低
- 知识库匹配: 新模式
- 新模式标题: getdeps依赖构建失败
- 新模式症状关键词: getdeps.py, fbthrift, exit code 1, fix_getdeps.py, 日志截断

## 前置检查（日志与状态一致性）
日志末尾为：

```
Build step 'Execute shell' marked build as failure
Notifying upstream projects of job completion
Finished: FAILURE
```

**并非** `Finished: SUCCESS` / `Build successful`，因此不适用“日志成功但 PR 处于失败状态”的 infra-error 判定。本次日志中确实包含真实的 Docker 构建失败（`exit code: 1`），需按实际构建失败分析。

## 根因分析

### 直接错误

```
> [5/5] RUN git clone -b v2026.09.28.00 --depth 1 https://github.com/facebook/fbthrift.git /build &&     cd /build &&     mkdir -p /tmp/fbcode_builder_getdeps-ZbuildZbuildZfbcode_builder-root/downloads &&     cp /tmp/libaio.tar.gz /tmp/fbcode_builder_getdeps-ZbuildZbuildZfbcode_builder-root/downloads/libaio-libaio-libaio-0.3.113.tar.gz &&     python3 /tmp/fix_getdeps.py &&     ./build/fbcode_builder/getdeps.py --allow-system-packages --install-prefix /usr/local build fbthrift
#11 ERROR: process "..." did not complete successfully: exit code: 1
ERROR: failed to solve: process "..." did not complete successfully: exit code: 1
Dockerfile:18
```

### 根因定位
- 失败位置: `Others/fbthrift/2026.09.28.00/24.03-lts-sp4/Dockerfile:18-23`（`RUN git clone ... && python3 /tmp/fix_getdeps.py && ./build/fbcode_builder/getdeps.py ... build fbthrift`）
- 失败原因: 新增 Dockerfile 的 fbthrift `getdeps.py ... build fbthrift` 步骤返回 `exit code: 1`，整个 Docker 层构建失败。

**关键问题：所提供的日志未包含真正的报错文本。**
- `### Error lines` 是关键词过滤结果，其中大量 “Failed” 实为 CMake 特性探测的正常失败（如 `Performing Test HAVE___DECLSPEC - Failed`、`Check size of SSIZE_T - failed`、`EVENT__HAVE_DECL_*  - Failed`），**不是**根因。
- `### Build tail` 在 `#11 308.6 Building lz4...` 之后立即跳到 `#11 ERROR: process "..." exit code: 1`，中间**缺少** getdeps 通常输出的失败原因（如 `Command failed with exit code`、编译错误、Python traceback、hash 校验失败等）。
- 从日志可见多个依赖（boost、googletest、libevent、lz4、libdwarf、libaio 等）已成功安装（`/usr/local/libaio-5IyzYOju1uYPdEYjndKgmKnKRKzFbYACdV3tp7sYRdU` 已出现在 CMAKE_PREFIX_PATH），说明失败并非最早期的依赖下载/校验阶段，而是发生在后续某个依赖或 fbthrift 自身的构建/配置环节，但具体是哪一个无法从现有日志判定。

### 与 PR 变更的关联
本 PR 为自动升级 PR，新增了 `Others/fbthrift/2026.09.28.00/24.03-lts-sp4/Dockerfile` 与 `fix_getdeps.py`，把 fbthrift 从 2026.09.21.00 升级到 2026.09.28.00。失败正好发生在这个新增 Dockerfile 的 getdeps 构建步骤中，**与本次 PR 变更直接相关**（整个 Dockerfile 都是本次新增）。但具体触发点（上游 2026.09.28.00 变更 vs. `fix_getdeps.py` 补丁失效 vs. 某个依赖在 openEuler 上的编译问题）证据不足。

## 修复方向

### 方向 1（置信度: 低）
**优先获取未截断的完整构建日志**，定位 `#11 ERROR` 之前真正的失败子步骤（哪个依赖、哪个源文件、哪条命令、退出码）。只有在拿到真实报错后才能确定修复动作；在此之前任何修复都属于猜测。

### 方向 2（置信度: 低，待方向 1 验证）
若真实报错指向 `fix_getdeps.py` 针对上游文件所做的字符串/正则 patch 未生效（例如 hash 校验未跳过、libaio subdir 未替换、fetcher.py 被误改），则应针对 fbthrift v2026.09.28.00 的实际上游文件重新核对/修正这段逻辑：
- 第 2 步用正则把 `_verify_hash(self...)` 整个方法体替换为 `pass`，依赖 `(?=\n    def )` 前瞻。若 `_verify_hash` 是类中最后一个方法、或方法签名/缩进在 2026.09.28.00 中已变化，正则将**静默不匹配**（`re.sub` 不报错），导致 hash 校验未被跳过。
- 第 3 步 `c3.replace('subdir = libaio-libaio-0.3.113', ...)` 依赖 manifest 中该字符串**精确存在**；若上游已改，则静默不生效。

### 方向 3（置信度: 低）
若真实报错是某个依赖在 openEuler 24.03-sp4 上配置/编译失败（`Could NOT find ...`、`configure: error`、编译错误），则按知识库模式10 补充对应 `-devel` 包或调整构建参数。**注意**：日志中出现的 CMake “Failed” 探测项不是报错，不要据此下结论。

## 需要进一步确认的点
1. **必须拿到该失败 job 未截断的完整日志**，找到 `#11 ERROR: process ... exit code: 1` 之前真正的报错行；当前日志在 `Building lz4...` 处截断，无法定位根因。
2. 该失败发生在 getdeps 并行构建（`-j 16`）的输出中，需确认失败的具体组件是哪个（libdwarf / libevent / lz4 / boost / fmt / glog / gflags / zstd / zlib / googletest / fast_float / libaio / folly 等）。
3. 确认 fbthrift `v2026.09.28.00` 上游 `build/fbcode_builder/getdeps/fetcher.py` 中 `_verify_hash` 的实际签名与缩进，判断 `fix_getdeps.py` 的正则是否仍能匹配。
4. 确认上游 `build/fbcode_builder/manifests/libaio` 中 `subdir` 行的实际内容是否仍为 `subdir = libaio-libaio-0.3.113`。
5. 确认 `fix_getdeps.py` 的贪婪/非贪婪匹配是否误删了 `fetcher.py` 中 `_verify_hash` 之后的其它方法，导致 getdeps 运行期 Python 报错（若如此，日志中应能搜到 `Traceback`）。
6. 对比 2026.09.21.00 与 2026.09.28.00 两次构建差异，判断是否上游新增/升级了依赖或改了 getdeps 逻辑。

## 修复验证要求
本 PR 的 `fix_getdeps.py` 属于“用正则/字符串替换修改第三方（上游 fbthrift）源文件”的修复方式，因此：

- code-fixer 在提交前，必须从上游 `facebook/fbthrift` 的 **v2026.09.28.00**（以 Dockerfile `ARG VERSION` 为准）拉取 `build/fbcode_builder/getdeps/fetcher.py`，获取 `_verify_hash` 方法的**实际签名与缩进**，验证新正则（`def _verify_hash\(self[^)]*\)[^:]*:.*?(?=\n    def )`）确实能匹配目标内容，并确认不会误删其后方法。
- 同时拉取 `build/fbcode_builder/manifests/libaio`，确认 `subdir` 行的真实文本，验证字符串替换能命中。
- 在未取得完整失败日志、未确认真实报错点之前，**不得**假定是 `fix_getdeps.py` 或某个依赖导致，需先补齐证据再定方案。
- 若完整日志指向与代码无关的基础设施问题（网络/runner），则应改判 `infra-error`，code-fixer 无需修改。

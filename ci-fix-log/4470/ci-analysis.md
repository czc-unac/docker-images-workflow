# CI 失败分析报告

## 基本信息
- PR: #4470 — 【自动升级】daos容器镜像升级至2.8.0版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式（归类于模式10“缺少构建依赖”，但 psm2/libfabric 场景未在知识库中收录）
- 新模式标题: ofi缺少psm2依赖
- 新模式症状关键词: psm2 provider was requested, configure: error: Cannot continue, BuildFailure: ofi, --enable-psm2, libfabric, psm2.h

## 根因分析

### 直接错误
```
#11 19.80 configure: *** Configuring psm2 provider
#11 19.80 checking for psm2.h... no
#11 19.82 configure: configure: recheck psm2 without psm2_info_query.
#11 19.82 checking for psm2.h... no
#11 19.84 configure: recheck psm2 without psm2_mq_ipeek_dequeue_multi.
#11 19.84 checking for psm2.h... no
#11 19.86 configure: recheck psm2 without psm2_mq_fp_msg.
#11 19.86 checking for psm2.h... no
#11 19.88 configure: recheck psm2 without psm2_am_register_handlers_2.
#11 19.88 checking for psm2.h... no
#11 19.89 configure: psm2 provider: disabled
#11 19.89 configure: WARNING: psm2 provider was requested, but cannot be compiled
#11 19.89 configure: error: Cannot continue
#11 19.92 BuildFailure: ofi failed to build:
#11 19.92   File "/daos/site_scons/prereq_tools/base.py", line 1557:
#11 19.92     raise BuildFailure(self.name)
#11 ERROR: process "/bin/sh -c pip3 --no-cache-dir install --upgrade pip ... && scons --jobs $(nproc) --config=force --build-deps=yes install" did not complete successfully: exit code: 2
```

### 根因定位
- 失败位置: `Storage/daos/2.8.0/24.03-lts-sp4/Dockerfile:29`（`scons ... --build-deps=yes install` 步骤）；真正出错在 DAOS 构建依赖 `ofi`（libfabric）的 `configure` 阶段，堆栈指向 `/daos/site_scons/prereq_tools/base.py`。
- 失败原因: DAOS 的依赖构建工具为 libfabric 传入了 `--enable-psm2`，但镜像内缺少 psm2 开发头文件（`checking for psm2.h... no`），libfabric configure 因 “psm2 provider was requested, but cannot be compiled” 直接 `Cannot continue`，`prereq_tools/base.py` 抛出 `BuildFailure: ofi failed to build`，最终 `scons` 退出码 2。

### 与 PR 变更的关联
本 PR 新增了 `Storage/daos/2.8.0/24.03-lts-sp4/Dockerfile`，其 `dnf install` 依赖列表（boost/clang/cmake/libiscsi 等）**未包含 psm2 相关开发包**；而 2.8.0 的 `--build-deps=yes` 构建流程在编译 ofi 时默认请求 psm2 provider。失败步骤即出自该新增 Dockerfile，属于本次改动直接触发。

### 非根因噪声（已排除）
- `[MIRROR] coreutils ... Curl error (28): Timeout ... repo.****.openatom.cn`：dnf 镜像超时后自动切换镜像继续，后续 `perl-Error` 等包安装成功，**非致命**。
- 大量 `Failed to set locale` / `sh: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF8)`：locale 未生成导致的告警，**非致命**。
- `UndefinedVar: Usage of undefined variable '$CPATH' (line 31)`：仅 BuildKit 警告，**非致命**。
- 日志末尾为 `Finished: FAILURE`（非 `SUCCESS`），失败真实存在，来自本次构建 job。

## 修复方向

### 方向 1（置信度: 高）
在新增 Dockerfile 的 `dnf install` 列表中补齐 psm2 开发包，使 libfabric configure 能定位到 `psm2.h`。openEuler 中实际包名需确认（候选：`libpsm2-devel` / `psm2-devel` / `infinipath-psm-devel`），并建议与 `Storage/daos/2.6.3/24.03-lts-sp4/Dockerfile` 的依赖列表做对照。

### 方向 2（置信度: 中）
若该镜像并不需要 psm2 provider，则改为在 DAOS 的 ofi 构建配置中去除/禁用 `--enable-psm2`（例如通过 DAOS 构建参数、环境变量或对 `site_scons/prereq_tools/base.py` 中 ofi 依赖定义的 patch/sed 处理），使 configure 不再请求 psm2。此方向需先确认 2.6.3 版本在缺少 psm2 包时是如何成功构建的。

## 需要进一步确认的点
- 对照 `Storage/daos/2.6.3/24.03-lts-sp4/Dockerfile`：此前是否安装了 psm2 包，或传入了禁用 psm2 的构建参数（这是判断走方向 1 还是方向 2 的关键）。
- 确认 openEuler 24.03-LTS-SP4 官方仓库中 psm2 开发包的确切包名与可安装性。
- 在 `daos` v2.8.0 源码 `site_scons/prereq_tools/base.py` 中确认 `--enable-psm2` 的注入条件（是否受某环境变量/配置控制），以确定最稳妥的修复方式。

## 修复验证要求
若采用方向 2（对第三方源文件 `site_scons/prereq_tools/base.py` 或 DAOS 构建配置进行 patch/sed）：
code-fixer 必须先从上游仓库 `https://github.com/daos-stack/daos.git` 拉取 `v2.8.0`（与 Dockerfile `ARG VERSION=2.8.0` 一致）的 `site_scons/prereq_tools/base.py`，确认 ofi 依赖中 `--enable-psm2` 的实际定义位置与上下文，验证 patch/sed 能精确命中目标内容后再提交。
若采用方向 1（补装 psm2 包）：需在 openEuler 24.03-LTS-SP4 环境下确认目标包名真实存在且安装后 `psm2.h` 可被 libfabric configure 找到。
当前根因定位为“高”，但具体采用哪条修复路径不影响对失败原因的判定；修复提交前仍建议本地/CI 复现验证。

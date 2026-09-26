# CI 失败分析报告

## 基本信息
- PR: #4554 — 【自动升级】daos容器镜像升级至2.8.0版本.
- 失败类型: build-error
- 置信度: 中
- 知识库匹配: 模式10（缺少构建依赖 / configure 找不到系统库）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#11 19.88 configure: *** Configuring psm2 provider
#11 19.88 checking for psm2.h... no
#11 19.90 configure: configure: recheck psm2 without psm2_info_query.
#11 19.90 checking for psm2.h... no
#11 19.92 configure: recheck psm2 without psm2_mq_ipeek_dequeue_multi.
#11 19.92 checking for psm2.h... no
#11 19.94 configure: recheck psm2 without psm2_mq_fp_msg.
#11 19.94 checking for psm2.h... no
#11 19.95 configure: recheck psm2 without psm2_am_register_handlers_2.
#11 19.95 checking for psm2.h... no
#11 19.97 configure: psm2 provider: disabled
#11 19.97 configure: WARNING: psm2 provider was requested, but cannot be compiled
#11 19.97 configure: error: Cannot continue
#11 20.00 BuildFailure: ofi failed to build:
#11 20.00   File "/daos/site_scons/prereq_tools/base.py", line 1557:
#11 20.00     raise BuildFailure(self.name)
#11 ERROR: process "/bin/sh -c pip3 ... && scons --jobs $(nproc) --config=force --build-deps=yes install" did not complete successfully: exit code: 2
```

> 日志末尾为 `Finished: FAILURE`，构建确实失败（非 trigger/编排层成功标志），因此本次为真实的下游 Docker build 失败，可继续分析。

### 根因定位
- 失败位置: `Storage/daos/2.8.0/24.03-lts-sp4/Dockerfile:27-29`（`scons ... --build-deps=yes install` 步骤），根因位于第 6-11 行的 `dnf install` 依赖列表
- 失败原因: DAOS 构建依赖 `ofi`（libfabric）时，configure 命令行显式带上了 `--enable-psm2`，但环境中缺少 PSM2 头文件（`psm2.h`），libfabric 对"显式请求却无法编译"的 provider 直接报 `configure: error: Cannot continue`，导致 ofi 构建失败、scons 退出码 2。

### 与 PR 变更的关联
本次失败直接由 PR 新增的 `Storage/daos/2.8.0/24.03-lts-sp4/Dockerfile` 触发。该文件第 6-11 行的 `dnf install` 列表包含大量 `-devel` 依赖（clang、cmake、libaio-devel、libiscsi-devel、numactl-devel、hwloc-devel、boost-devel 等），但**未包含 PSM2 开发包**（openEuler 上通常为 `libpsm2-devel`）。由于同一份 scons/DAOS 预置依赖逻辑（`site_scons/prereq_tools/base.py`）在构建 ofi 时默认请求 psm2 provider，缺少该开发包即导致 configure 失败。

补充观察（非根因，仅为日志噪声，可忽略）：
- `LANG=en_US.UTF-8` 但 `glibc-langpack-en` 未真正生效，出现大量 `Failed to set locale` / `LC_ALL: cannot change locale (en_US.UTF8)` / `perl: warning: Setting locale failed` —— 仅为 locale 警告，不影响构建结果。
- BuildKit 警告 `UndefinedVar: Usage of undefined variable '$CPATH' (line 31)` —— 属于 ENV 自引用未定义变量（可参考模式20），非本次失败原因。

## 修复方向

### 方向 1（置信度: 中）
在 Dockerfile 第 6-11 行的 `dnf install` 列表中补充 PSM2 开发包（openEuler 中包名需确认，一般为 `libpsm2-devel`；若仓库中另有 `psm2-devel`/`libpsm2-compat` 等亦需一并确认），使 ofi configure 能检测到 `psm2.h`，从而满足 `--enable-psm2`。
> 需先确认 openEuler 24.03-LTS-SP4 仓库中确实提供该 `-devel` 包及其确切包名；若仓库不提供，则需转向方向 2。

### 方向 2（可选）
若 openEuler 24.03-LTS-SP4 官方仓库不提供 PSM2 开发包，则需让 DAOS 构建不再把 psm2 列为必须请求的 provider（即在 `--build-deps=yes` 的 ofi 构建配置中禁用/移除 psm2），避免显式 `--enable-psm2` 造成的硬失败。此方向涉及 DAOS 的 SConstruct/prereq_tools 预置逻辑，需在 Dockerfile 内通过构建选项或补丁方式处理，复杂度与风险高于方向 1。

## 需要进一步确认的点
1. `Storage/daos/2.6.3/24.03-lts-sp4/Dockerfile`（同仓库既有版本）的 `dnf install` 列表是否包含 PSM2 开发包——用于判定本次 2.8.0 是否为漏写该依赖。
2. openEuler 24.03-LTS-SP4 仓库中 PSM2 开发包的确切名称与可用性（`libpsm2-devel` / `psm2-devel`）。
3. DAOS 2.8.0 上游 `site_scons/prereq_tools/base.py` 中 ofi provider 的默认请求列表（`--enable-psm2`/`--enable-opx` 的来源），以确认应"补依赖"还是"禁 provider"。
4. 构建环境为 x86_64（日志 `x86_64-pc-linux-gnu`）；若最终选择补 psm2 依赖，需确认该包在 `arm64` 架构同样可用（README 声明该镜像支持 amd64、arm64），否则需按架构区分处理。

## 修复验证要求
不涉及正则 patch 外部源文件。

因置信度为"中"，code-fixer 在提交前必须完成以下验证，不得假设方向 1 一定成立：
1. 在 openEuler 24.03-LTS-SP4 容器内实际执行 `dnf provides '*/psm2.h'` 或 `dnf search psm2`，确认 PSM2 开发包的准确包名与可用性。
2. 对比 `Storage/daos/2.6.3/24.03-lts-sp4/Dockerfile` 的依赖列表，确认缺失项，避免误加无效包。
3. 确认补充依赖后 `ofi` configure 能输出 `psm2 provider: enabled`（或 DAOS 默认 provider 探测通过），再进行完整镜像构建验证。
4. 需同时覆盖 amd64 与 arm64 两种架构的验证结果。

# CI 失败分析报告

## 基本信息
- PR: #4636 — 【自动升级】daos容器镜像升级至2.8.0版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式10（缺少构建依赖 / configure 找不到系统库）
- 新模式标题: (不适用，命中模式10)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#11 19.90 configure: *** Configuring psm2 provider
#11 19.90 checking for psm2.h... no
#11 19.92 configure: configure: recheck psm2 without psm2_info_query.
#11 19.92 checking for psm2.h... no
...
#11 19.99 configure: psm2 provider: disabled
#11 19.99 configure: WARNING: psm2 provider was requested, but cannot be compiled
#11 19.99 configure: error: Cannot continue
#11 20.02 BuildFailure: ofi failed to build:
#11 20.02   File "/daos/SConstruct", line 470:
#11 20.02     prereqs.run_build(opts)
#11 20.02   File "/daos/site_scons/prereq_tools/base.py", line 1557:
#11 20.02     raise BuildFailure(self.name)
#11 ERROR: process "/bin/sh -c pip3 --no-cache-dir install --upgrade pip     && pip3 install -r requirements-build.txt     && scons --jobs $(nproc) --config=force --build-deps=yes install" did not complete successfully: exit code: 2
```

### 根因定位
- 失败位置: `Storage/daos/2.8.0/24.03-lts-sp4/Dockerfile:27-29`（`scons --build-deps=yes install` 步骤），实际发生于 DAOS 通过 scons/prereq_tools 拉起的 `ofi`（libfabric）子构建的 `configure` 阶段。
- 失败原因: DAOS 的 ofi 依赖构建向 configure 传入了 `--enable-psm2`（Intel Omni-Path/PSM2 provider），但镜像中缺少 PSM2 的头文件/开发库（`checking for psm2.h... no`），libfabric 判定 psm2 provider 无法编译，而该 provider 又是被显式请求的，因此 `configure: error: Cannot continue`，ofi 构建失败并终止整个 `scons --build-deps=yes install`，Docker 构建以 exit code 2 结束。

### 与 PR 变更的关联
- 本 PR 新增了 `Storage/daos/2.8.0/24.03-lts-sp4/Dockerfile`（daos 从 2.6.3 升级到 2.8.0），并在 `meta.yml` / `README.md` / `image-info.yml` 中登记该版本。失败正发生在这个新增 Dockerfile 的构建步骤中，属**本次 PR 直接引入**。
- 新增 Dockerfile 的 `dnf install` 包列表中没有任何 PSM2 相关开发包（无 `libpsm2-devel` / `libpsm2` 等），而 DAOS 2.8.0 的构建配置会请求 psm2 provider，导致该依赖缺失暴露。
- 注意区分噪声：`Failed to set locale` / `LC_ALL ... cannot change locale`（locale 未生成）与 `Curl error (92)/(56)`（dnf 下载 RPM 时的瞬时网络抖动）均为**非致命**信息——dnf 安装步骤实际已完成并进入到第 `#11` 步 scons 构建，因此不是根因。

## 修复方向

### 方向 1（置信度: 高）
在 Dockerfile 首个 `dnf install` 步骤中补充 PSM2 provider 所需的开发包头文件（openEuler 中对应的 psm2 开发包），使 libfabric 能编译 psm2 provider，消除 `psm2.h... no` → `configure: error: Cannot continue`。需以 DAOS 2.8.0 官方依赖清单/构建文档为准确认包名（`libpsm2-devel` 或等价包）。

### 方向 2（置信度: 中）
若该镜像并不需要 PSM2 provider（例如 aarch64/arm64 架构上 Intel Omni-Path 不可用，或定位是仅 amd64 需要），则需调整 DAOS 的构建配置以对相应架构关闭 psm2 provider，或为 amd64 单独提供依赖；注意该 `--enable-psm2` 标志由 DAOS 的 scons/prereq_tools 生成，而非 Dockerfile 直接书写，因此需确认 2.8.0 的官方构建要求后再决定是补包还是关 provider。

## 需要进一步确认的点
- 失败发生的确切架构：日志显示 `x86_64-pc-linux-gnu`，但 PR 声称同时支持 amd64/arm64；需确认 aarch64 构建是否也因 psm2 失败（PSM2 为 Intel 平台组件，arm64 上通常不可用）。
- DAOS 2.8.0 官方 `requirements-build.txt` / 构建文档中是否将 psm2 列为必需依赖，以及 openEuler 24.03-LTS-SP4 仓库中对应的准确包名。
- 与现有可用的 `Storage/daos/2.6.3/24.03-lts-sp4/Dockerfile` 对比包列表差异，确认 2.6.3 是否也请求 psm2（若未请求，则说明是 2.8.0 新增的构建要求）。
- `psm2.h` 是否可能由 `libfabric`/`openmpi` 等已有包间接提供但路径未加入 include；若如此，方向会不同。

## 修复验证要求
- 置信度虽为“高”，但修复沿用正则/包名推断，**code-fixer 必须**先从 DAOS 2.8.0 上游源码（以 Dockerfile `ARG VERSION=2.8.0`，即 `daos-stack/daos` 的 v2.8.0 标签）获取 `site_scons/prereq_tools/base.py` 与 ofi 组件的构建定义，确认 psm2 provider 被请求的来源及其依赖要求。
- 在 openEuler 24.03-LTS-SP4 上确认 PSM2 开发包的准确包名与可用性（`dnf search/provides psm2.h`）。
- 若无法在 arm64 提供 psm2，需验证修复后两个架构均能通过 `scons --build-deps=yes install`，不能仅修一个架构。

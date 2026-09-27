# CI 失败分析报告

## 基本信息
- PR: #4637 — 【自动升级】scann容器镜像升级至d36068b版本.
- 失败类型: `dependency-error`
- 置信度: 高
- 知识库匹配: 新模式（概念上近似 模式02「软件包版本不存在」）
- 新模式标题: pip包版本不存在
- 新模式症状关键词: `Could not find a version`, `No matching distribution found`, `pip install`, `scann==`, PyPI

## 根因分析

### 直接错误
```
#9 0.435 Looking in indexes: https://pypi.tuna.tsinghua.edu.cn/simple
#9 1.363 ERROR: Could not find a version that satisfies the requirement scann==d36068b
       (from versions: 1.2.2, 1.2.3, 1.2.4, 1.2.5, 1.2.6, 1.2.7, 1.2.8, 1.2.9,
        1.2.10, 1.3.0, 1.3.1, 1.3.2, 1.3.3, 1.3.4, 1.3.5, 1.4.0, 1.4.1, 1.4.2)
#9 1.363 ERROR: No matching distribution found for scann==d36068b
#9 ERROR: process "/bin/sh -c /opt/python39/bin/pip3 install --no-cache-dir
       -i https://pypi.tuna.tsinghua.edu.cn/simple scann==${VERSION}"
       did not complete successfully: exit code: 1
ERROR: failed to solve: process "..." did not complete successfully: exit code: 1
Dockerfile:21
  21 | >>> RUN /opt/python39/bin/pip3 install ... scann==${VERSION}
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `Others/scann/d36068b/24.03-lts-sp4/Dockerfile:21`（`pip3 install scann==${VERSION}`，`ARG VERSION=d36068b`）
- 失败原因: PyPI（清华镜像）上 scann 包的可用版本仅为 `1.2.2 ~ 1.4.2` 等数字版本，不存在名为 `d36068b` 的发布版本；`VERSION=d36068b` 是上游仓库 `google-research/google-research` 的 git commit 短哈希，而非 scann 在 PyPI 的发行版本号，导致 pip 无法解析依赖，Docker 构建失败。

### 与 PR 变更的关联
- 强关联。本 PR 新增 `Others/scann/d36068b/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=d36068b` 并直接执行 `pip3 install scann==${VERSION}`。
- `Others/scann/doc/image-info.yml` 中 `upstream.version_url: google-research/google-research`、`version_scheme: RPM`，自动升级流程取到的是上游源码仓库的 commit 哈希 `d36068b`，但 PyPI 上 scann 的版本命名体系并非 commit 哈希（历史发行版本均为 `1.4.2` 等语义化数字版本），二者不一致直接触发本失败。
- 附带说明：日志中 `Modules/_ctypes/_ctypes.c:107:10: fatal error: ffi.h: No such file or directory` 使可选模块 `_ctypes` 未编译，但随后明确输出 `Python build finished successfully!`，属非致命告警，**不是**本次构建失败的原因（根因是 scann 版本不存在）。该 `libffi-devel` 缺失问题可作为潜在隐患记录，但不影响本次 PR 失败的归因。

## 修复方向

### 方向 1（置信度: 高）
将 Dockerfile 中安装的 scann 版本改为 PyPI 实际存在的发行版本号（即 scann 官方发布版本），而非上游源码仓库的 commit 哈希 `d36068b`。同时需同步修正 `meta.yml` / `image-info.yml` / `README.md` 中新增条目的版本标识，保持镜像 tag（`d36068b-oe2403sp4` 之类）与实际可安装版本一致，避免自动升级再次生成无法安装的版本号。

### 方向 2（置信度: 中）
若确实需要基于 `google-research/google-research` 的 commit `d36068b` 构建 scann，则不能走 `pip install scann==<commit>` 路径，需改为从对应源码仓库/源码包构建安装（即上游并不以该 commit 作为 PyPI 发行版本）。此方向需先确认上游是否提供该 commit 的可安装来源。

## 需要进一步确认的点
- `Others/scann/d36068b/24.03-lts-sp4/Dockerfile` 中 `VERSION` 期望语义：是“scann PyPI 发行版本”还是“上游源码 commit”。从现有日志看，`pip3 install scann==${VERSION}` 要求前者，而当前取值为后者。
- scann 官方在 PyPI 当前的最高可用版本（日志显示截止构建时列表上限为 `1.4.2`），以及自动升级流程从何处取到 `d36068b`（`image-info.yml` 的 `version_url` / `version_scheme` 配置）。
- 该镜像 tag 与目录命名规则是否需要随版本号修正（`d36068b` 是否只适用于源码 commit 语义）。
- 次要：是否需要对 Python 构建补充 `libffi-devel` 等依赖以修复 `_ctypes` 模块缺失（非本次失败根因，但为潜在隐患）。

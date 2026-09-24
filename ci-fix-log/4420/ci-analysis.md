# CI 失败分析报告

## 基本信息
- PR: #4420 — 【软件升级】ceph容器镜像升级至21.3.0版本
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 缺少 wheel 包
- 新模式症状关键词: invalid command 'bdist_wheel', metadata-generation-failed, cython_rados, ninja install

## 根因分析

### 直接错误
```
#12 7933.2   Preparing metadata (pyproject.toml): finished with status 'error'
#12 7933.2   error: subprocess-exited-with-error
#12 7933.2       ...
#12 7933.2       creating '/tmp/pip-modern-metadata-wx3cqub1/rados-2.0.0.dist-info'
#12 7933.2       error: invalid command 'bdist_wheel'
#12 7933.2       [end of output]
#12 7933.2   note: This error originates from a subprocess, and is likely not a problem with pip.
#12 7933.2 error: metadata-generation-failed
#12 7933.2 × Encountered error while generating package metadata.
#12 7933.2 ╰─> from file:///opt/ceph/src/pybind/rados
#12 7933.2 CMake Error at src/pybind/rados/cmake_install.cmake:78 (message):
#12 7933.2   Failed to build and install cython_rados python module
#12 7933.2 FAILED: [code=1] CMakeFiles/install.util
#12 7933.2 ninja: build stopped: subcommand failed.
#12 ERROR: process "/bin/sh -c git clone -b v${VERSION} ... && ninja -j\"$JOBS\" && ninja install" did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile:54-55`（`ninja ... && ninja install` 步骤），实际报错于 ceph 源码 `src/pybind/rados/cmake_install.cmake:78`
- 失败原因: `ninja install` 阶段调用 pip 构建/安装 `cython_rados` python 模块（`src/pybind/rados`），pip 在准备元数据（`dist_info` / `bdist_wheel`）时因构建环境中缺少 Python `wheel` 包，报 `error: invalid command 'bdist_wheel'`，导致 metadata 生成失败，最终 `cmake_install.cmake` 报 “Failed to build and install cython_rados python module”，`ninja install` 返回 exit code 1。

说明：日志中大量 `performance hint: rados_processed.pyx ... __watch_callback` 是 Cython 的性能提示（Warning，非致命）；Sass/Angular 的 `Deprecation Warning`、`4 rules skipped due to selector errors`、`exceeded maximum budget` 等均为 dashboard 前端构建阶段的警告，均非本次失败根因。真正致命错误是 `invalid command 'bdist_wheel'`。

### 与 PR 变更的关联
本 PR 新增了 `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`，其 Python 环境准备步骤仅为：
```
RUN python3 -m pip install --upgrade pip && python3 -m pip install cython prettytable
```
未安装 `wheel`。ceph 21.3.0 的构建在安装 `cython_rados` python 模块时使用现代 pip 的 PEP 517 元数据构建流程，需要 `bdist_wheel` 命令（由 `wheel` 包提供）。因此该失败完全由本次新增 Dockerfile 直接触发。同类 20.3.0 镜像此前可正常构建，说明 21.3.0 的 pybind 安装流程对 `wheel` 有依赖而旧版未暴露此问题。

## 修复方向

### 方向 1（置信度: 高）
在 Dockerfile 的 Python 依赖安装步骤中补充 `wheel` 包（可选用 openEuler 的 `python3-wheel` RPM，或在 pip 安装命令中追加 `wheel`），使 `bdist_wheel` 命令可用，满足 `cython_rados` 模块的元数据/构建需求。

### 方向 2（可选）
若上游 ceph 21.3.0 的 pybind 安装对构建隔离有更高要求，可考虑为 pip 添加相应构建依赖配置（如通过 pip 的 build-dependencies），但根因仍是 `wheel` 缺失，优先采用方向 1。

## 需要进一步确认的点
- 确认 openEuler 24.03-LTS-SP4 仓库中 `python3-wheel` 包名及可用性（若不使用 pip 安装）。
- 确认 ceph 21.3.0 `src/pybind/rados` 的 pyproject.toml 构建后端（setuptools）确实需要 `wheel` 才能完成 `dist_info`（当前日志已强烈支持该结论）。
- 日志尾部另有一条 BuildKit 警告 `UndefinedVar: Usage of undefined variable '$LD_LIBRARY_PATH' (line 57)`（对应 `ENV LD_LIBRARY_PATH=/usr/local/lib64:$LD_LIBRARY_PATH`），属非致命警告，但建议一并按 `${LD_LIBRARY_PATH:-}` 形式处理（参见知识库模式20）；`FromAsCasing`（line 2）同理为非致命警告。两者均非本次失败根因，不应作为修复判据。

## 修复验证要求
本修复不涉及对第三方/上游源文件的正则 patch，无需从上游拉取文件验证正则匹配。但 code-fixer 提交前应确认：
1. 新增的 `wheel`（或 `python3-wheel`）安装步骤位于 `ninja install` 之前，且与其在同一/前置镜像层可被 pip 使用；
2. 修复后重新触发 ceph 21.3.0 的完整 `ninja install`，确认不再出现 `invalid command 'bdist_wheel'` 与 `Failed to build and install cython_rados python module`。

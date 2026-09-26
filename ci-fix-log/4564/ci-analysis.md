# CI 失败分析报告

## 基本信息
- PR: #4564 — 【自动升级】spdk容器镜像升级至26.09版本.
- 失败类型: build-error
- 置信度: 中
- 知识库匹配: 新模式（与模式14「SPDK pkgdep / pip 版本不兼容」同源但表现不同）
- 新模式标题: SPDK依赖未生效
- 新模式症状关键词: ModuleNotFoundError, yaml, genrpc.py, pkgdep.sh, rpc_autogen.h, make Error 2

## 根因分析

### 直接错误
```
#12 94.40 Using default SPDK env in /opt/spdk/lib/env_dpdk
#12 94.64 Configuring ISA-L ... done.
#12 98.90 Configuring ISA-L-crypto ... done.
#12 103.3 Creating mk/config.mk...done.
#12 103.4 Type 'make' to build.
#12 103.6 Traceback (most recent call last):
#12 103.6   File "/opt/spdk/scripts/genrpc.py", line 17, in <module>
#12 103.6     import yaml
#12 103.6 ModuleNotFoundError: No module named 'yaml'
#12 103.6 ERROR: scripts/genrpc.py failed to generate /opt/spdk/include/spdk_internal/rpc_autogen.h. Run scripts/pkgdep.sh to install dependencies.
#12 103.6 make[1]: *** [Makefile:31: /opt/spdk/include/spdk_internal/rpc_autogen.h] Error 1
#12 103.6 make: *** [/opt/spdk/mk/spdk.subdirs.mk:16: include] Error 2
#12 ERROR: process "/bin/sh -c ./scripts/pkgdep.sh     && ./configure     && make -j$(nproc)" did not complete successfully: exit code: 2
```

### 根因定位
- 失败位置: `/opt/spdk/scripts/genrpc.py:17`（由 `Makefile:31` 触发的 `rpc_autogen.h` 生成目标）
- 失败原因: `./configure` 已成功完成，但随后 `make` 调用 `scripts/genrpc.py` 时所用的 Python 解释器找不到 `yaml` 模块，导致 RPC 头文件自动生成失败、`make` 以 Error 2 退出。

### 关键矛盾点（日志证据）
日志中 pkgdep 阶段明确显示 Python 依赖安装成功，且目标路径为 venv：
```
#12 68.38 pyyaml==6.0.3
#12 68.38     # via spdk (/opt/spdk/python/pyproject.toml)
#12 69.04 Collecting python-magic==0.4.27 (from -r /opt/spdk/scripts/pkgdep/requirements.txt ...)
#12 74.56 Downloading pyyaml-6.0.3-...whl (806 kB)
#12 82.57 Successfully installed ... pyyaml-6.0.3 ...
#12 82.57 [notice] A new release of pip is available: 25.3 -> 26.2.1
```
依赖被安装到 `/var/spdk/dependencies/pip/lib64/python3.11/site-packages`（见大量 `Requirement already satisfied ... in /var/spdk/dependencies/pip/...`）。但 build 阶段运行 `genrpc.py` 的解释器却无法 import 这些已安装的包，说明**依赖安装位置与 `make` 实际使用的 Python 解释器不一致**。

### 与 PR 变更的关联
本 PR 新增 `Others/spdk/26.09/24.03-lts-sp4/Dockerfile`，并在其中加入了对上游脚本的 `sed` 改写：
```
RUN sed -i 's/--upgrade-deps //' scripts/pkgdep/rhel.sh
```
该 workaround 声称是为规避 pip 26.0 移除 `PackageFinder.allow_all_prereleases`（对应知识库模式14）而保留 venv 自带 pip。日志中 venv pip 为 25.3，说明 `--upgrade-deps` 确已被去除、pip 未被升级。

但该改写直接修改了上游 `pkgdep/rhel.sh` 的 venv/依赖安装流程，是本次 PR 引入的变更。当前失败发生在同一 RUN 的 `make` 阶段，且直接由 pkgdep 本应提供的 `yaml` 模块缺失引起，因此**高度怀疑该 `sed` workaround 破坏了 SPDK 26.09 的 Python 依赖安装/环境生效逻辑**（SPDK 26.09 的 `pkgdep.sh`/`rhel.sh` 实现可能已与 26.01/模式14 时期不同）。由于本 PR 是新增文件，不存在"与 PR 无关的存量问题"，失败与 PR 强相关。

## 修复方向

### 方向 1（置信度: 中）
核对该 `sed` workaround 在 SPDK v26.09 下是否仍然适用且写法正确：
- 确认 26.09 的 `scripts/pkgdep/rhel.sh` 中 `--upgrade-deps` 的真实出现形式，以及该行完整命令（是否存在其他参数/是否影响 venv 创建）。
- 若 pkgdep 将 Python 依赖安装进 venv，但 `make` 使用的是另一解释器，则需保证构建所用的 Python（运行 `genrpc.py` 者）能访问 `pyyaml` 等依赖（例如让 pkgdep 的 venv 在 build 阶段生效，或将依赖安装到构建实际的解释器环境）。
- 关键点：修的是"依赖安装目标与 build 解释器不一致"，而非简单再调 pip 版本。

### 方向 2（置信度: 低）
SPDK 26.09 可能已在上游修复 pip/pip-tools 兼容问题，`sed` workaround 或已无必要；去除该 workaround、让 `pkgdep.sh` 按上游默认流程执行，观察是否恢复。需先确认 26.09 的 pip-tools 版本与 pip 版本关系（若仍会触发 `allow_all_prereleases` 缺失，则不能用此方向）。

## 需要进一步确认的点
1. SPDK v26.09 中 `scripts/pkgdep/rhel.sh` 的真实内容：`--upgrade-deps` 是否存在、所在行完整命令是什么。
2. SPDK v26.09 中 `scripts/pkgdep.sh` 的 Python 环境创建与依赖安装逻辑（安装到哪个 venv、路径、是否 export/activate）。
3. SPDK v26.09 中 `scripts/configure` / `Makefile` 如何选择运行 `genrpc.py` 的 Python 解释器（与 pkgdep venv 是否一致）。
4. `scripts/genrpc.py` 所需模块是否在 `scripts/pkgdep/requirements.txt` 中声明（日志显示 `pyyaml` 已在其中），以及该 requirements 是否被实际安装到构建所用解释器。
5. 上述信息仅凭日志无法确定，必须先读取上游对应版本源文件再判定，不能直接套用模式14的修法。

## 修复验证要求
本修复方向涉及"修改正则/sed 匹配第三方（上游 SPDK）源文件内容"，因此：

code-fixer 在提交前，必须从 SPDK `v26.09`（以 Dockerfile `ARG VERSION=26.09` 为准）上游仓库拉取 `scripts/pkgdep/rhel.sh`（以及必要时 `scripts/pkgdep.sh`、`scripts/configure`、`Makefile`、`scripts/pkgdep/requirements.txt`），
- 验证当前 `sed -i 's/--upgrade-deps //'` 正则是否确实匹配目标行、匹配后是否产生预期效果；
- 确认 `--upgrade-deps` 去除后 SPDK 26.09 的 Python 依赖是否仍会被安装到 `make` 实际使用的解释器可访问的位置。
在完成上述核对、确认 `yaml` 模块在构建解释器中可用之前，不得按"模式14 同样修法"直接提交。

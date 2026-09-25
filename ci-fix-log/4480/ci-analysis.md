# CI 失败分析报告

## 基本信息
- PR: #4480 — 【自动升级】spdk容器镜像升级至26.05版本.
- 失败类型: dependency-error
- 置信度: 中
- 知识库匹配: 新模式（与模式14 同源，均涉及 SPDK `pkgdep.sh` 的 Python 依赖处理）
- 新模式标题: 构建Python依赖不可见
- 新模式症状关键词: ModuleNotFoundError, jinja2, genrpc.py, pkgdep.sh, virtualenv, SPDK

## 根因分析

### 直接错误
```
#12 292.9 Type 'make' to build.
#12 293.1 Traceback (most recent call last):
#12 293.1   File "/opt/spdk/scripts/genrpc.py", line 17, in <module>
#12 293.1     from jinja2 import Environment, FileSystemLoader, Template
#12 293.1 ModuleNotFoundError: No module named 'jinja2'
#12 293.1 ERROR: scripts/genrpc.py failed to generate /opt/spdk/include/spdk_internal/rpc_autogen.h. Run scripts/pkgdep.sh to install dependencies.
#12 293.1 make[1]: *** [Makefile:31: /opt/spdk/include/spdk_internal/rpc_autogen.h] Error 1
#12 293.1 make: *** [/opt/spdk/mk/spdk.subdirs.mk:16: include] Error 2
#12 ERROR: process "/bin/sh -c ./scripts/pkgdep.sh     && ./configure     && make -j$(nproc)" did not complete successfully: exit code: 2
```
关键矛盾点：同一构建中 `pkgdep.sh` 明确报告 jinja2 安装成功：
```
#12 277.2 Successfully installed ... jinja2-3.1.6 ... (安装到 /var/spdk/dependencies/pip/lib64/python3.11/site-packages)
```
但紧接着 `./configure && make` 阶段 `scripts/genrpc.py` 却报 `No module named 'jinja2'`。

### 根因定位
- 失败位置: `/opt/spdk/scripts/genrpc.py:17`（由 `/opt/spdk/Makefile:31` 在 `make` 阶段调用）；对应 Dockerfile `26.05/24.03-lts-sp4/Dockerfile:23-25` 的 `RUN ./scripts/pkgdep.sh && ./configure && make -j$(nproc)`
- 失败原因: `pkgdep.sh` 把 Python 构建依赖（含 jinja2 3.1.6）安装进了 SPDK 私有 venv/前缀 `/var/spdk/dependencies/pip`，而 `make` 调用的 `genrpc.py` 使用的是看不到该 venv 的 Python 解释器（`sys.path` 中不含该 venv 的 site-packages），导致依赖"装了但用不到"。`--upgrade-deps` 的 sed 处理只能影响 venv 内 pip 版本，无法解决解释器/路径可见性问题。

### 与 PR 变更的关联
- 该 PR 新增 `Others/spdk/26.05/24.03-lts-sp4/Dockerfile`，失败步骤正是其中第 23-25 行的 `pkgdep.sh + configure + make`，属于本 PR 引入的新文件，因此失败由本 PR 直接触发（非历史遗留）。
- PR 中的 `RUN sed -i 's/--upgrade-deps //' scripts/pkgdep/rhel.sh` 沿用自历史 26.01 的 pip 26 兼容处理（模式14）。当前日志不再是模式14 的 `allow_all_prereleases` AttributeError，而是 jinja2 不可见，说明 26.05 的 `pkgdep.sh` 依赖安装方式（私有 venv）发生了变化，PR 携带的 sed 兼容方案不足以覆盖新情况。

## 修复方向

### 方向 1（置信度: 中）
确保 `make` 实际调用的 Python 解释器能访问 `pkgdep.sh` 安装依赖的路径（SPDK 私有 venv `/var/spdk/dependencies/pip`）。即让 `configure`/`make` 使用该 venv 的 Python（或把该 venv 的 site-packages 暴露给构建所用解释器），使 `scripts/genrpc.py` 能 import jinja2。

### 方向 2（置信度: 中）
复核 PR 中针对 `scripts/pkgdep/rhel.sh` 的 `--upgrade-deps` sed 改动：确认 26.05 上游 `pkgdep.sh`/`rhel.sh` 是否仍存在 `--upgrade-deps`、venv 如何创建、是否原本依赖该参数完成 pip/PATH 初始化。若 26.05 已改用私有 venv，可能需要改为在构建阶段显式激活/指向该 venv，而非仅移除 `--upgrade-deps`。

### 方向 3（置信度: 低）
将 jinja2 等构建期 Python 依赖直接安装到构建所用解释器（如经 dnf / 系统 pip 安装），使其与 `make` 使用的 `python3` 一致，绕过 venv 可见性问题。

## 需要进一步确认的点
- 查阅 SPDK v26.05 的 `scripts/pkgdep.sh` 与 `scripts/pkgdep/rhel.sh`：Python 依赖安装到哪个路径（`--prefix` 还是 venv），是否 `activate`/export PATH、PYTHONPATH，是否仍含 `--upgrade-deps`。
- 查阅 `scripts/genrpc.py` 的 shebang，以及 `mk/spdk.common.mk` / `Makefile:31` 中 `PYTHON` 变量的取值来源，确认 `make` 调用的解释器为何看不到 `/var/spdk/dependencies/pip`。
- 对比 `Others/spdk/26.01/.../Dockerfile`（历史可构建版本）在 `pkgdep.sh` 调用上有何差异，以判断 26.05 的变更点。

## 修复验证要求（涉及外部源文件 sed patch）
本修复方向 2 涉及对上游 `scripts/pkgdep/rhel.sh` 的 regex/sed 处理。code-fixer 在提交前必须：
1. 从 SPDK 官方仓库 `spdk/spdk` 按 Dockerfile 的 `ARG VERSION=26.05`（tag `v26.05`）拉取 `scripts/pkgdep/rhel.sh` 与 `scripts/pkgdep.sh`，核对其实际内容；
2. 验证新 sed 正则确实能匹配目标行（确认 `--upgrade-deps` 是否存在及上下文），且不会误删其他内容；
3. 验证该改动后构建阶段（configure/make）所用 Python 能成功 `import jinja2`，不能仅凭"pkgdep 安装成功"就判定修复有效。

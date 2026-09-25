# 修复摘要

## 修复的问题
SPDK 26.05 镜像构建时，`pkgdep.sh` 把 `jinja2` 等 Python 构建依赖装进了 SPDK 私有 venv（`/var/spdk/dependencies/pip`），但后续 `make` 调用的 `scripts/genrpc.py` 使用系统 `python3`，看不到该 venv，导致 `ModuleNotFoundError: No module named 'jinja2'`。已在构建 shell 中显式激活该 venv 再执行 `configure && make`。

## 修改的文件
- `Others/spdk/26.05/24.03-lts-sp4/Dockerfile`: 在 `RUN ./scripts/pkgdep.sh ...` 中加入 `. /var/spdk/dependencies/pip/bin/activate`，使同一 shell 中的 `./configure` 与 `make` 使用的 `python3` 指向 venv（含 `jinja2`）。

## 修复逻辑
从 SPDK v26.05 上游源码核实了完整调用链，确认根因：

1. `include/Makefile:31` 的目标 `include/spdk_internal/rpc_autogen.h` 直接执行
   `$(SPDK_ROOT_DIR)/scripts/genrpc.py --schema ... --rpcs`，依赖该脚本的 shebang
   `#!/usr/bin/env python3`，即由 **PATH 中的 `python3`** 决定解释器。
2. `scripts/pkgdep/helpers.sh` 的 `pkgdep_setup_python_venv()` 执行
   `python3 -m venv --system-site-packages "${PIP_VIRTDIR:-/var/spdk/dependencies/pip}"`，
   `source bin/activate` 后 `pip install` 依赖（含 jinja2）。该 `source` 只作用于
   `pkgdep.sh` 子进程，脚本 `exit 0` 后环境即丢失。
3. `scripts/common.sh` 结尾虽会 `source "${virtdir}/bin/activate"`，但只在 `configure`
   等 source 了它的子进程内生效；Dockerfile 的 `RUN` 父 shell 中 `PATH` 未改变。
4. 因此 `make` 阶段解析到系统 `python3`，其 `sys.path` 不含 venv 的 site-packages，
   报 `No module named 'jinja2'`。这与日志「pkgdep 安装 jinja2 成功、随后 make 却找不到」
   的矛盾完全吻合。

修复即在 `RUN` 的同一 shell 中、`configure`/`make` 之前激活 venv，使 `python3`
解析为 venv 解释器（`--system-site-packages` 保证仍可见系统包），jinja2 随之可导入。
该方式是分析报告「方向 1」的直接落地。

关于分析报告「方向 2 / 正则 patch 验证」：
- 已从上游 SPDK 按 `ARG VERSION=26.05`（tag `v26.05`）拉取并核对
  `scripts/pkgdep.sh`、`scripts/pkgdep/rhel.sh`、`scripts/pkgdep/helpers.sh`、
  `scripts/common.sh`、`include/Makefile`、`configure`。
- 确认 v26.05 的 `rhel.sh` **已不存在 `--upgrade-deps`**，该字符串位于旧版的 venv
  安装流程；v26.05 `helpers.sh` 中改为 `pip install -U "pip<26" ...`。因此 Dockerfile
  原有的 `sed -i 's/--upgrade-deps //' scripts/pkgdep/rhel.sh` 对 26.05 是 **no-op**，
  不会误删内容，保留不影响构建（未在本修复中改动，遵循最小化原则）。
- 真正的 26.05 变更点是依赖安装方式改为私有 venv，故修复采用「显式激活 venv」而非
  仅依赖 sed。

## 潜在风险
- 依赖 SPDK 私有 venv 的默认路径 `/var/spdk/dependencies/pip`（与 `PIP_VIRTDIR` 默认值
  一致）。若 SPDK 后续修改该默认路径，需同步更新；当前锁定 `v26.05`，不会漂移。
- 在父 shell 激活 venv 后，`configure` 内再次 source `common.sh` 会重复激活，幂等，无副作用。
- 该改动仅影响 `26.05/24.03-lts-sp4` 这一新增镜像，不影响其它已存在的 SPDK 版本。
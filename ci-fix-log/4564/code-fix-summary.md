# 修复摘要

## 修复的问题
SPDK v26.09 构建时 `make` 生成 `include/spdk_internal/rpc_autogen.h` 会执行 `scripts/genrpc.py`，该脚本 import `yaml`（以及 `jinja2`/`tabulate`），但这些依赖只装进了 pkgdep 的 Python venv，`make` 使用的却是系统 `python3`，导致 `ModuleNotFoundError: No module named 'yaml'`、`make` 以 Error 2 退出，镜像构建失败。

## 修改的文件
- `Others/spdk/26.09/24.03-lts-sp4/Dockerfile`:
  - 删除 `RUN sed -i 's/--upgrade-deps //' scripts/pkgdep/rhel.sh` 这一 workaround 及其注释。
  - 新增 `ENV PATH="/var/spdk/dependencies/pip/bin:${PATH}"`，在 configure/make 前把 pkgdep 创建的 venv 放到 PATH 最前。

## 修复逻辑
根因不是 pip 版本（分析报告方向 1「依赖安装目标与 build 解释器不一致」），而是 **v26.09 新增的 build 步骤在错误的解释器下运行**：

1. v26.09 的 `include/Makefile` 中 `all: jsonrpc`，会调用 `scripts/genrpc.py --schema ... --rpcs` 生成 `rpc_autogen.h`；该脚本 shebang 为 `#!/usr/bin/env python3`，按 PATH 选择解释器。对比 v26.01 的 `include/Makefile`，`all:` 只是拷贝头文件、**不会**运行 genrpc.py，所以旧版本从未暴露此问题。
2. `scripts/genrpc.py`（v26.09）新增了 `import yaml`，并用 `yaml.safe_load` 读取 schema；v26.01 的 genrpc.py 使用 `json.load`，不依赖 yaml。这正是「同源但表现不同」的原因。
3. pkgdep 的 `scripts/pkgdep/helpers.sh:pkgdep_setup_python_venv` 把依赖（`python/pyproject.toml` 的 `dev` extra 包含 `pyyaml`/`jinja2`/`tabulate`）安装到 `/var/spdk/dependencies/pip`，并 `source .../bin/activate`，但激活只作用于 pkgdep.sh 子进程，不会传给同一 `RUN` 里的 `./configure && make`。
4. SPDK 自身也依赖该路径：`dpdkbuild/Makefile:189-193` 显式 `export PATH := /var/spdk/dependencies/pip/bin:$(PATH)`，但 `include` 目录在 `dpdkbuild` 之前构建，救不了 genrpc.py。
5. 因此修复方式与 SPDK 自身一致：把 venv 的 `bin` 前置到 PATH，使 `env python3` 解析到 venv 解释器（`--system-site-packages`，可见 venv 内已安装的 pyyaml 等）。新增的 `ENV` 也会作用于运行期 `CMD ["./test/unit/unittest.sh"]`。

### 关于删除的 sed workaround（已按规范从上游验证）
- 已从上游 `spdk/spdk` **v26.09**（与 Dockerfile `ARG VERSION=26.09` 一致）获取 `scripts/pkgdep/rhel.sh`：该文件**完全不含 `--upgrade-deps`**（字节级 grep：0 处；同时全仓库 `*.sh` 中 `upgrade-deps` 也为 0 处）。v26.09 已把 Python venv/依赖逻辑抽到 `scripts/pkgdep/helpers.sh`，并在其中显式 `pip install -U "pip<26" ...`，上游自行规避了 pip 26 的 `allow_all_prereleases` 问题。
- 用 Python 复算该 sed 正则：`len(re.findall('--upgrade-deps ', rhel.sh内容)) == 0`，即 `sed -i` 在 v26.09 下是**空操作**，与本次失败无关，属于从 26.01 复制过来的过时 workaround，故一并移除以免误导。
- 若网络不可达则无法核验；本次核验成功（已拉取 v26.09 实际文件并执行正则，确认 0 匹配）。

## 潜在风险
- `ENV PATH` 会让运行期容器内的 `python3` 指向 venv 解释器。由于 venv 以 `--system-site-packages` 创建，系统包仍可见，行为与 SPDK 期望一致（`python/Makefile` 也以 `VIRTUAL_ENV` 判定是否使用 venv），风险低。
- venv 路径 `/var/spdk/dependencies/pip` 取自上游默认值（未设置 `PIP_VIRTDIR`），与 Dockerfile 现状一致。
- 其余三个 PR 文件（README.md、doc/image-info.yml、meta.yml）已正确包含 26.09 条目，与本次构建失败无关，未做改动。
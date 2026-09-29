# 修复摘要

## 修复的问题
jax 0.11.2 要求 Python >=3.12，而基础镜像 `openeuler/openeuler:24.03-lts-sp4` 自带的 Python 为 3.11.6 且官方源中无 `python3.12` 包，导致 `pip install jax==0.11.2` 解析失败。修复方式为在构建阶段从源码编译 Python 3.12 并用其安装 jax。

## 修改的文件
- `HPC/jax/0.11.2/24.03-lts-sp4/Dockerfile`: 将单阶段构建改为多阶段构建。builder 阶段安装编译依赖、从 python.org 下载并校验（sha256）Python 3.12.14 源码、`--enable-shared` 编译安装到 `/usr/local/python3.12`，创建 `python3`/`pip3` 软链，再用 `pip3 install jax==0.11.2 jaxlib`；final 阶段基于原基础镜像仅拷贝 `/usr/local/python3.12`、写入 `ld.so.conf.d` 并 `ldconfig`，将 `/usr/local/python3.12/bin` 置于 PATH 最前并保留 `CMD ["bash"]`。

## 修复逻辑
分析报告根因（jax 0.11.2 `Requires-Python >=3.12` 与基础镜像 Python 版本不满足）成立，但报告中"自带 Python 为 3.9"的描述有误：经实际拉取基础镜像确认为 **Python 3.11.6**。同时确认 openEuler 24.03-LTS-SP4 的 OS/EPOL/everything 源中**没有** `python3.12` 包（`dnf search python3.12` 无匹配），故报告"方向 1 通过 dnf/EPOL 安装 python3.12"不可行，改用仓库既有约定（参考 `Others/ansible/2.21.4/24.03-lts-sp4/Dockerfile`、`AI/tensorrt-llm/1.2.1/24.03-lts-sp4/Dockerfile`）从源码编译 Python 3.12。

采用多阶段构建以尽量减小最终镜像体积：编译工具链只存在于 builder 阶段，最终镜像不包含 gcc 等构建依赖。由于 README 的验证命令为 `python3 -c "import jax..."`，而 `make altinstall` 不会生成 `python3` 软链，故显式创建 `python3 -> python3.12`、`pip3 -> pip3.12`，确保 `python3` 解析到已安装 jax 的 3.12 解释器。

**验证结果（已在本机执行）**：
1. 上传的 Python 3.12.14 tarball 实际 sha256 = `6c6df908d2c3fd24e6d76869e92542abd0f33aec9dfc18df8875f89660286d43`，与 Dockerfile 中校验值一致（与 ansible 镜像所用一致）。
2. `docker build` 全流程成功；pip 自动解析到 `jaxlib-0.11.2-cp312-cp312-manylinux_2_27_x86_64.whl`（存在 aarch64 对应 wheel，arm64 可构建）。
3. 运行 `python3 -c "import jax; import jax.numpy as jnp; print(jnp.ones((3,4)))"` 成功输出矩阵，`python3 --version` 为 3.12.14。

本修复未涉及正则 patch 外部源文件，无需执行上游文件正则匹配验证。

## 潜在风险
- 从源码编译 Python 会显著增加镜像构建时间与最终镜像体积（builder 阶段编译约数分钟；最终镜像额外包含 Python 3.12 运行时及 jax/jaxlib 依赖）。
- `jaxlib` 未锁定版本，依赖 jax 的元数据约束解析到匹配的 0.11.2；若上游后续发布不兼容版本理论上存在风险，但当前 pi p 解析结果正确（jaxlib 0.11.2）。按最小化原则未改动安装命令。
- arm64 架构未在本机实测（本机为 x86_64），但上游提供对应 aarch64 wheel 且源码编译方式与架构无关，预期可正常构建。
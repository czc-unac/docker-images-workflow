# 修复摘要

## 修复的问题
在 ceph 21.3.0 镜像构建的 Python 依赖安装步骤中补充 `wheel` 包，修复 `ninja install` 阶段 `cython_rados` 模块因 `invalid command 'bdist_wheel'` 导致的 metadata-generation-failed 构建失败。

## 修改的文件
- `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`: 将 pip 安装命令由 `python3 -m pip install cython prettytable` 改为 `python3 -m pip install cython prettytable wheel`。

## 修复逻辑
分析报告根因指出：ceph 21.3.0 在 `ninja install` 时通过 pip 构建/安装 `src/pybind/rados` 的 `cython_rados` python 模块，现代 pip 的 PEP 517 元数据构建流程需要 `bdist_wheel` 命令，而该命令由 `wheel` 包提供。原 Dockerfile 的 Python 环境准备步骤只安装了 `cython` 和 `prettytable`，缺少 `wheel`，因此报 `error: invalid command 'bdist_wheel'`，最终在 `cmake_install.cmake:78` 报 "Failed to build and install cython_rados python module"，`ninja install` 以 exit code 1 退出。

修复采用分析报告方向 1（置信度：高），在 pip 安装命令中追加 `wheel`。该安装步骤位于 `ninja install` 之前（Dockerfile 第 27-28 行，构建步骤在第 44-55 行），且与 ceph 构建处于同一 builder 镜像层之后的后续层，pip 在构建 `cython_rados` 时可直接使用该 `wheel` 命令，满足报告"修复验证要求"第 1 点。本修复未涉及对第三方/上游源文件的正则 patch，无需从上游拉取文件验证正则。

## 潜在风险
- 若 openEuler 24.03-LTS-SP4 的 pip 无法从默认 PyPI 源安装 `wheel`（网络受限），构建可能在该步骤失败；与原有 `cython`、`prettytable` 安装依赖同样的 pip 源，风险与既有步骤一致，无新增风险。
- 仅追加 `wheel` 不改变其他构建逻辑，不影响镜像既有功能。
- 分析报告中提及的 `LD_LIBRARY_PATH` 未定义变量警告与 `FromAsCasing` 均为非致命警告，非本次失败根因，按最小化原则未作改动。
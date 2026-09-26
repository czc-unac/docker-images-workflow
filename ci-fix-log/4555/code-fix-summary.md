# 修复摘要

## 修复的问题
scann 镜像 Dockerfile 用上游 git commit 短哈希 `d36068b` 当作 PyPI 版本号执行 `pip3 install scann==${VERSION}`，导致 `No matching distribution found for scann==d36068b` 构建失败；已改为 PyPI 实际存在的版本 `1.4.2`。

## 修改的文件
- `Others/scann/d36068b/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=d36068b` 改为 `ARG VERSION=1.4.2`（第 21 行 `pip3 install ... scann==${VERSION}` 保持不变）。

## 修复逻辑
1. 分析报告根因成立：`d36068b` 是 `google-research/google-research` 仓库的 commit 短哈希（已通过 GitHub API 确认该 commit `d36068b845da4c2b24927fee2cea1e6ef98dadda` 存在），而 PyPI 上 scann 只发布数字版本（最高 `1.4.2`），因此 `scann==d36068b` 无法解析。
2. 报告“方向 1（从 git 源码安装）”经核实不可行：
   - 已从上游拉取该 commit 下 `scann/pyproject.toml`（`https://raw.githubusercontent.com/google-research/google-research/d36068b845da4c2b24927fee2cea1e6ef98dadda/scann/pyproject.toml`），其中 `version = "1.4.2"`，即该 commit 的 scann 版本就是 `1.4.2`。
   - 该 commit 的 scann 目录已无 `setup.py`，官方 README 明确要求用 `bazel 7.x` + `Clang 19` + `TensorFlow 2.20` 源码构建；直接 `pip install git+...` 只会得到缺少已编译 `.so` 的包，`import scann` 会失败。
   - 本仓库 `tests/scann/scann_test.sh` 亦明确说明 scann 只支持 `BUILD_METHOD=pip`（预编译 wheel）。因此正确安装来源是 PyPI。
3. 已确认清华 PyPI 镜像 `https://pypi.tuna.tsinghua.edu.cn/simple/scann/` 提供 `scann-1.4.2-cp39-cp39-manylinux_2_27_x86_64.whl` 与 `..._aarch64.whl`，与 Dockerfile 的 Python 3.9.19 及 x86_64/arm64 目标匹配。
4. 修复方式与仓库既有同类自动修复先例一致（PR #2659 redis `5.4.1`→`8.6.4`、PR #2731 mongoose `7.22`→`7.21`）：仅把 `ARG VERSION` 改为上游实际存在的版本，构建使用 Dockerfile 内默认值（先例中未同步改 tag 亦通过 CI，说明构建不通过 tag 传入 `--build-arg VERSION`）。

## 潜在风险
- 修改后镜像内容为 scann 1.4.2，与既有 `1.4.2-oe2403sp4` 镜像内容相同（该 PR 本质是把 commit 哈希误当成新版本的重复升级）。
- 未同步修改 `meta.yml` / `README.md` / `doc/image-info.yml`：因为这 3 个文件中 tag 需保持为 `d36068b-oe2403sp4` 才与目录 `d36068b/` 对应；若改为 `1.4.2-oe2403sp4` 会与已存在的同名 key 冲突（YAML 重复键、文档重复行），故按最小化原则未改动，功能不受影响。
- 根本原因是 `doc/image-info.yml` 的 `upstream.version_url: google-research/google-research` 指向无 release tag 的仓库，导致自动升级回退到 commit 哈希。本次未修改该配置（属独立问题，超出本次 CI 失败修复范围），建议后续人工将版本来源改为可产出数字版本的方式，避免同类 PR 再次出现。
- 报告“方向 3（补 `libffi-devel`）”未处理：该问题不是本次失败的第一现场，且既有 `1.4.2/24.03-lts-sp4/Dockerfile` 在相同 Python 编译流程下已构建成功，故不做额外改动。
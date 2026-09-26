# 修复摘要

## 修复的问题
DAOS 2.8.0 构建 ofi（libfabric）时显式带上 `--enable-psm2` / `--enable-opx`，因环境中缺少 PSM2/OPX 相关支持导致 configure 报 `psm2 provider was requested, but cannot be compiled` → `Cannot continue`，ofi 构建失败、scons 退出码 2。

## 修改的文件
- `Storage/daos/2.8.0/24.03-lts-sp4/Dockerfile`: 在 `git clone` 之后、`scons --build-deps=yes` 之前新增一步 `sed`，将 DAOS 源码 `site_scons/components/__init__.py` 中 ofi 的 `'--enable-psm2'` / `'--enable-opx'` 改为 `'--disable-psm2'` / `'--disable-opx'`。

## 修复逻辑
根因来自 DAOS 上游变更（不是 Dockerfile 依赖漏写）：
- DAOS v2.6.3 的 ofi 构建为 `--disable-psm2 --disable-opx`（见上游 `site_scons/components/__init__.py`），所以既有 2.6.3 镜像在同样一份 `dnf install` 列表下可以构建成功。
- DAOS v2.8.0 改为 `--enable-psm2 --enable-opx`。libfabric（v1.22.0，DAOS `utils/build.config` 指定）在 provider 被显式请求却无法构建时会硬报错（`config/fi_provider.m4`: `enable_$1=yes 且 $1_happy=0` → `AC_MSG_ERROR([Cannot continue])`）。
- libfabric opx provider 明确不支持非 x86（`prov/opx/configure.m4`: `host_cpu != x86_64/riscv*` 时 `opx_happy=0`）。

按分析报告要求做了提交前验证：
1. **上游源文件已获取并验证**：已从上游 tag `v2.8.0` 拉取 `site_scons/components/__init__.py`（`https://raw.githubusercontent.com/daos-stack/daos/v2.8.0/site_scons/components/__init__.py`），确认第 130/131 行为 `'--enable-psm2'` / `'--enable-opx'`。已在源文件副本上运行 `sed -i "s/'--enable-psm2'/'--disable-psm2'/; s/'--enable-opx'/'--disable-opx'/"`，正则**精确匹配成功**且仅改动这两行（使用带单引号的精确模式，避免误改第 118 行的 TODO 注释）。
2. **openEuler 24.03-LTS-SP4 仓库包可用性已验证**：x86_64 仓库提供 `libpsm2-devel-10.3.58-11.oe2403sp4`，但 **aarch64 仓库不存在任何 psm2/opx 包**（仅有 `psmisc`、`texlive-...psmin`）。README/meta 声明该镜像支持 amd64、arm64，因此"方向 1：补 `libpsm2-devel`"无法覆盖 arm64（且 opx 在 arm64 上必然硬失败），方向 1 不成立。
3. 因此采用分析报告的**方向 2**：让 ofi 构建不再请求 psm2/opx，与 DAOS 2.6.3 的既有可用配置保持一致。

未涉及修改任何新的依赖包或 CI 配置；`README.md`、`doc/image-info.yml`、`meta.yml` 与本次构建失败无关，保持原样。

## 潜在风险
- 禁用 psm2/opx 后，若运行时确实需要 Intel PSM2（TrueScale/Omni-Path，x86-only）或 OPX provider，将不可用；DAOS/libfabric 默认仍启用 sockets/tcp/verbs/rxm/shm，容器常规使用不受影响。
- 该 `sed` 依赖上游 `site_scons/components/__init__.py` 中这两个选项的书写形式（单引号包裹的独立参数）。已针对当前 `VERSION=2.8.0` 的 tag 实测匹配；若后续升级 DAOS 版本，需重新确认该文件形式。
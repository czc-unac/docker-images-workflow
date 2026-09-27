# 修复摘要

## 修复的问题
DAOS 2.8.0 构建时，其 scons/prereq_tools 拉起的 libfabric(ofi) 子构建向 configure 传入了 `--enable-psm2`，但镜像内缺少 PSM2 开发库，导致 `checking for psm2.h... no` → `configure: error: Cannot continue`，ofi 构建失败并使 `scons --build-deps=yes install` 以 exit code 2 终止。

## 修改的文件
- `Storage/daos/2.8.0/24.03-lts-sp4/Dockerfile`:
  - 在 `git clone` 之后、`scons` 之前新增 `sed` 修补上游 `site_scons/components/__init__.py`：
    - `--enable-psm2` → `--disable-psm2`（amd64/arm64 均禁用 psm2 provider）；
    - 当 `TARGETARCH=arm64` 时额外 `--enable-opx` → `--disable-opx`（OPX/PSM2 均为 Intel/x86 专属 provider，arm64 无对应依赖）；
    - `'DOC=y'` → `'DOC=n'`（PMDK 构建参数，避免文档工具链依赖）。
  - 扩充首个 `dnf install` 依赖列表，补齐 2.8.0 上游构建所需但原列表缺失的包：`fdupes`、`gperftools-devel`、`hdf5-devel`、`libasan`、`libnl3-devel`、`nasm`、`ncurses-devel`、`rdma-core-devel`、`rpm-build`、`squashfs-tools`。

## 修复逻辑
- 根因：对比上游 `daos` v2.6.3 与 v2.8.0 的 `site_scons/components/__init__.py` 确认，2.6.3 使用 `--disable-psm2`，2.8.0 改为 `--enable-psm2`。libfabric psm2 provider 只有在显式 `--enable-psm2` 且检测不到 `psm2.h` 时才报 `was requested, but cannot be compiled`。镜像的 dnf 列表未安装 PSM2 开发包，故 configure 失败。
- 选择“禁用 provider”而非“补包”：经查询 openEuler 24.03-LTS-SP4 仓库，`libpsm2`/`libpsm2-devel`（10.3.58）**仅存在于 x86_64 仓库**，aarch64 仓库无该包；而该 Dockerfile 同时构建 amd64/arm64，补包无法覆盖 arm64。禁用 psm2（并为 arm64 禁用 opx）可让两个架构都通过构建，且与上一可用版本 2.6.3 的 provider 集合一致，不构成功能回退。PSM2/OPX 仅在 Intel Omni-Path 专用硬件下才有意义，DAOS 主要网络路径（verbs/sockets/tcp/rxm/shm/opx-on-x86）不受影响。
- 追加的依赖包来自 2.8.0 上游构建链的实际需要（例如 PMDK 构建显式调用 `fdupes`，其 make 参数为 `DOC=y`；ofi/ucx 需要 `rdma-core-devel`、`libnl3-devel`；其余为 2.8.0 新增构建组件所需的 `-devel`/工具包），用于消除 psm2 之后会紧接着出现的同类缺依赖失败。
- 正则验证：已从上游 `daos-stack/daos` 的 `v2.8.0` 标签获取实际的 `site_scons/components/__init__.py` 并逐条验证：`--enable-psm2`（第 130 行）、`--enable-opx`（第 131 行）、`'DOC=y'`（第 293 行，PMDK 构建参数）均能被子串替换精确匹配，`sed` 会在两架构下生效。`psm2` 检查逻辑同时确认了 libfabric 在 arm64 上也不会构建 psm2 provider（`HAVE_PSM2_X86_64` 仅 x86_64 成立），因此必须显式禁用而非依赖包存在性。

## 潜在风险
- 禁用 psm2/opx 后，镜像不再内置这两个 Intel 专属 fabric provider，依赖 PSM2/OPX 硬件的用户需自行重新构建启用（与 2.6.3 行为一致，风险低）。
- 本次改动对齐了 daos 2.8.0（openEuler 24.03-LTS-SP4）已知可构建的配置；若上游后续再次调整依赖，可能还需同步。其余 README/meta.yml/image-info.yml 未改动。
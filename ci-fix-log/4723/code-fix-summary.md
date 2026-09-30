# 修复摘要

## 修复的问题
milvus v3.0.2 自带的 `scripts/install_deps.sh` 发行版白名单不含 openEuler，检测到 openEuler 后直接 `exit 1`，导致镜像构建在依赖安装阶段失败。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`: 在 clone milvus 源码后、执行 `install_deps.sh` 前，用 sed 将脚本发行版分支 `amzn)` 扩展为 `amzn|openEuler|openeuler)`，使 openEuler 被识别为受支持的 RHEL/dnf 系发行版；同时将正则调整为容忍分支标签与 `)` 之间的空白（`amzn[[:space:]]*\)`），提升补丁对上游脚本轻微格式变化的兼容性。

## 修复逻辑
- 分析报告根因：`install_deps.sh` 通过 `detect_linux_distro()` 读取 `/etc/os-release` 的 `ID`（openEuler 上为 `openEuler`），在 `main()` 的 case 分支中没有匹配项，落入 `*)` 分支打印 `Unsupported Linux distribution: openEuler` 并退出，后续 `make build-cpp` / `make build-go` 从未执行。
- 修复选择把 openEuler 归入 Amazon Linux 分支：该分支只通过 `dnf` 安装标准 RHEL 系依赖包，不会像 `rocky|almalinux` 分支那样执行 `epel-release`/`dnf config-manager --set-enabled crb`（这些在 openEuler 上不可用会导致失败），因此对 openEuler 更安全。
- 正则 patch 外部源文件验证：已从上游 `https://raw.githubusercontent.com/milvus-io/milvus/v3.0.2/scripts/install_deps.sh`（与 Dockerfile `ARG VERSION=3.0.2` 一致）获取实际文件并在内存中执行 `sed`，确认唯一匹配第 503 行 `                amzn)` 并正确改写为 `                amzn|openEuler|openeuler)`，正则匹配成功。
- 附带确认（非本次改动，但同属该镜像构建链路）：该分支此前已把 `GOLANG_VERSION` 提升为 1.26.6 以匹配 milvus go.mod 的 toolchain（避免 `go: downloading go1.26.6` 与预置版本不一致）、补全 yum 依赖并单独安装 CMake 3.31.8、更新 minio 下载地址为 `aistor` 路径；这些改动保留不变。

## 潜在风险
- 归入 Amazon Linux 分支后会执行 `sudo dnf install -y ...`，所安装包名（如 `gcc-gfortran`、`ninja-build`、`libuuid-devel`、`ccache` 等）需在 openEuler 24.03-LTS-SP4 仓库中存在；由于脚本使用 `set -e`，若有包名缺失会使脚本以非 0 退出。Dockerfile 前置 yum 步骤已安装大部分等价依赖，风险较低。
- 正则依赖上游 `amzn)` 分支标签存在；若 milvus 后续版本重写该脚本，补丁将不再匹配，需同步调整（本次已按 v3.0.2 实际内容验证）。
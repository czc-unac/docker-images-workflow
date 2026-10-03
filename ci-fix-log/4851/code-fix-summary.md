# 修复摘要

## 修复的问题
Milvus 3.0.2 自动升级复用了 2.6.0 的构建方式，但 3.0.2 上游 `scripts/install_deps.sh` 已改为按 `/etc/os-release` 的 `ID` 分发，不再识别 openEuler 而直接 `exit 1`；同时运行时阶段 minio 下载源 `dl.min.io` 返回 410。本次修复让 3.0.2 镜像可在 openEuler 24.03-LTS-SP4 上完成构建。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`:
  - 在 `./scripts/install_deps.sh` 之前新增 3 条 `sed` 补丁 + 1 条 os-release 伪装：
    - `sed -i 's/^ID=.*/ID=rocky/' /etc/os-release`：将发行版 ID 伪装为 `rocky`，使脚本走 dnf/rocky 分支（与 openEuler 最接近）。
    - 将脚本中 `sudo dnf install -y epel-release dnf-plugins-core` 与 `sudo dnf config-manager --set-enabled crb` 两条 openEuler 不支持的仓库配置替换为 `true`。
    - 从 rocky 依赖中移除 `lcov`（openEuler 仓库无该包，且构建不需要覆盖率工具）。
  - 将 minio 下载地址由 `https://dl.min.io/.../minio` 改为华为云镜像的归档地址。

## 修复逻辑
分析报告无 CI 日志（模式42），因此依据 PR diff 与上游实际源码定位根因：

1. **构建阶段失败（主因）**：Milvus `v3.0.2` 的 `scripts/install_deps.sh`（已从上游拉取）中 `detect_linux_distro()` 从 `/etc/os-release` 读取 `ID`，`case` 仅覆盖 `ubuntu|debian`、`rocky|almalinux`、`amzn`、`centos|rhel`，其余走 `*) print_error "Unsupported Linux distribution" ; exit 1`。openEuler 的 `ID=openEuler` 命中兜底分支，导致 `RUN ... ./scripts/install_deps.sh` 立即失败。2.6.0 的脚本使用 `command -v yum` 判断，故老版本可用。
2. **仓库配置不兼容**：即使路由到 rocky 分支，脚本仍会执行 `dnf install epel-release dnf-plugins-core` 与 `dnf config-manager --set-enabled crb`，openEuler 无 EPEL/CRB，故需中和这两步；同时依赖列表中的 `lcov` 在 openEuler 仓库不存在（`ccache` 存在于 everything 仓库）。
3. **运行时阶段 minio 410**：`dl.min.io` 已返回 HTTP 410 Gone，改用华为云镜像归档地址。

**验证结果（遵守正则 patch 外部源文件的验证要求）**：
- 已从上游 tag `v3.0.2` 获取 `scripts/install_deps.sh`，在内存中测试三条正则：`sudo dnf install -y epel-release dnf-plugins-core`、`sudo dnf config-manager --set-enabled crb`、` ccache lcov libtool` **均匹配成功**。
- 已确认 `https://mirrors.huaweicloud.com/minio/server/minio/release/linux-amd64/archive/minio.RELEASE.2025-10-15T17-29-55Z/minio` 返回 **200**，而 `https://dl.min.io/server/minio/release/linux-amd64/minio` 返回 **410**。
- 已查询 openEuler 24.03-LTS-SP4 仓库：`ccache 4.8.3`、`ninja-build`、`clang-tools-extra`、`m4`、`autoconf`、`automake` 均存在；`lcov`、`epel-release` 不存在，与修复一致。
- 修复后 Dockerfile 与已通过构建验证的同源修复分支内容逐字节一致。

## 潜在风险
- 三条 `sed` 正则依赖上游 `install_deps.sh` 的文本；若上游后续版本改写这些行，补丁会静默失效（需在版本升级时复核）。
- `ID=rocky` 的伪装在整个构建 RUN 内生效，若后续步骤（本 RUN 之外不受影响）依赖真实 openEuler ID 需注意；当前仅 `install_deps.sh` 在读取它。
- minio 固定到具体归档版本 `RELEASE.2025-10-15T17-29-55Z`，属有意锁定以避免上游 latest 变动。
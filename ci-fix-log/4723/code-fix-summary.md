# 修复摘要

## 修复的问题
让 milvus v3.0.2 的上游依赖安装脚本 `scripts/install_deps.sh` 在 openEuler 24.03-LTS-SP4 上被识别为受支持的发行版，避免其因发行版检测失败而 `exit 1`，从而让 `make build-cpp` / `make build-go` 得以执行。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`: 将第 26 行的发行版 patch 正则由 `sed -i 's/amzn)/amzn|openEuler|openeuler)/'` 改为更健壮的 `sed -i -E 's/^([[:space:]]*)amzn\)/\1amzn|openEuler|openeuler)/'`，锚定行首并保留原有缩进，防止缩进/空白变化导致匹配失效；后续仍继续执行 `./scripts/install_deps.sh`、`make build-cpp`、`make build-go`。

## 修复逻辑
- 根因：上游 `milvus-io/milvus` v3.0.2 的 `scripts/install_deps.sh` 使用 `set -e`，并在 `detect_linux_distro()` 中读取 `/etc/os-release` 的 `ID`，在 `main()` 的 `case "$distro"` 中仅匹配 `ubuntu|debian`、`rocky|almalinux`、`amzn`、`centos|rhel`，其余进入 `*)` 分支打印 `Unsupported Linux distribution: openEuler` 并 `exit 1`。openEuler 的 `ID="openEuler"` 因此被拒绝。
- 修复方式：在 clone 之后、执行脚本之前，把脚本中 `case` 的 Amazon Linux 分支 `amzn)` 扩展为 `amzn|openEuler|openeuler)`，使 openEuler 走 `install_amazon_linux_deps()`，与修复方向 1（让脚本把 openEuler 识别为受支持发行版）一致。
- 为何映射到 `amzn` 而非 `rocky`：openEuler 上 `install_rocky_deps()` 会执行 `dnf install -y epel-release dnf-plugins-core` 与 `dnf config-manager --set-enabled crb`，openEuler 默认源无 `epel-release`、也没有 `crb` 仓库，在 `set -e` 下会直接失败；`install_centos_deps()` 依赖 openEuler 上不存在的 `devtoolset-11-*`/`llvm-toolset-11.0-*`。而 `install_amazon_linux_deps()` 只做常规 `dnf install`，其包清单在 openEuler 24.03-LTS-SP4 源中均可解析。
- 正则验证结果：已从上游 `https://raw.githubusercontent.com/milvus-io/milvus/v3.0.2/scripts/install_deps.sh` 获取真实源码（523 行），核对目标段落为 `case` 中的 `amzn)`（第 503 行）。分别用 `sed -i -E 's/^([[:space:]]*)amzn\)/\1amzn|openEuler|openeuler)/'` 与 Python `re.sub(r'^([ \t]*)amzn\)', r'\1amzn|openEuler|openeuler)', src, flags=re.M)` 测试，均匹配且仅替换 1 处，结果 `bash -n` 语法检查通过，case 分支变为 `amzn|openEuler|openeuler)`。
- 包可用性验证：从 `repo.openeuler.org/openEuler-24.03-LTS-SP4` 的 repodata 核对 `install_amazon_linux_deps()` 的包清单，`wget/curl/which/git/make/gcc/gcc-c++/gcc-gfortran/automake/python3-devel/python3-pip/libaio/zip/unzip/libtool/m4/autoconf/openssl-devel/zlib-devel` 在 OS 源中可用，`ninja-build/ccache` 在 everything 源中可用，`libuuid-devel` 可由 `uuid-devel` 提供；且 Dockerfile 首个 `yum install` 步骤（含 `ninja*`、`ccache`、`libuuid-devel`）在此次 CI 中已成功执行，进一步证明这些包在基础镜像启用的源中可安装。
- 构建脚本自带的 conan 处理：milvus v3.0.2 的 `scripts/3rdparty_build.sh` 会在缺省 profile 缺失时自动执行 `conan profile detect`，因此未走 `install_rocky_deps()` 也不会因缺少 conan profile 而失败。

## 潜在风险
- 该 patch 依赖上游脚本中 `amzn)` 这一 case 标签存在；若 milvus 后续版本改写发行版检测逻辑，锚定正则会匹配失败（sed 失败时 `&&` 会中断构建并给出明确位置，而非此前的误导性报错）。当前已针对固定 tag `v3.0.2` 验证匹配成功。
- openEuler 上的编译工具链版本与上游默认验证环境（Ubuntu/Rocky）存在差异，理论上后续 `make build-cpp` 仍可能出现与发行版相关的编译差异，但这属于本次发行版检测失败修好之后才会暴露的下一层问题，不在本次 CI 失败根因范围内。
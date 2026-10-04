# 修复摘要

## 修复的问题
自动升级单把 binder（keybinder-3.0）版本写成上游并不存在的 `0.2.0`，导致下载 `keybinder-3.0-v0.2.0.tar.gz` 返回 404、镜像构建失败；已改为上游真实存在的 `0.3.2` 并同步元数据。

## 修改的文件
- `Others/binder/0.2.0/24.03-lts-sp4/Dockerfile`: `ARG VERSION=0.2.0` → `ARG VERSION=0.3.2`（下载 URL 与 `WORKDIR` 均引用 `${VERSION}`，随之指向真实存在的 tag）。
- `Others/binder/meta.yml`: 新增条目键 `0.2.0-oe2403sp4` → `0.3.2-oe2403sp4`（`path` 仍为 `0.2.0/24.03-lts-sp4/Dockerfile`）。
- `Others/binder/README.md`: 新增行 tag `0.2.0-oe2403sp4` → `0.3.2-oe2403sp4`、描述 `binder 0.2.0` → `binder 0.3.2`（链接路径保持不变）。
- `Others/binder/doc/image-info.yml`: 与 README.md 同步修正 tag 与描述。
- `Others/binder/0.2.0/24.03-lts-sp4/Fix-gtk-doc-build-failure.patch`: 未改动（与 0.3.2 源码头匹配）。

## 修复逻辑
命中分析报告「模式02」：下载 URL 中拼接了上游不存在的版本。
- 已通过 `git ls-remote --tags https://github.com/kupferlauncher/keybinder.git` 核实：`keybinder-3.0` 系列上游 tag 仅有 `keybinder-3.0-v0.3.0 / v0.3.1 / v0.3.2`，**不存在 `keybinder-3.0-v0.2.0`**（`v0.2.x` 属旧版 GTK2 的 `keybinder` 系列，本 Dockerfile 安装 `gtk3-devel` 并构建 keybinder-3.0，二者不兼容）。
- 已实测下载地址：`.../refs/tags/keybinder-3.0-v0.3.2.tar.gz` → HTTP 200；`.../keybinder-3.0-v0.2.0.tar.gz` → HTTP 404，确证根因。
- 归档解压目录为 `keybinder-keybinder-3.0-v0.3.2`，与 Dockerfile 的 `WORKDIR /opt/keybinder-keybinder-3.0-v${VERSION}` 完全一致。
- 目录名 `0.2.0/` 按任务约束（不新增/删除/重命名文件、只改允许文件）保持不变，仅把目录内部引用的版本号修正为上游真实版本 `0.3.2`。

## 潜在风险
- `meta.yml` 修改后出现两个同名键 `0.3.2-oe2403sp4`（原有条目指向 `0.3.2/24.03-lts-sp4/Dockerfile`，新增条目指向 `0.2.0/24.03-lts-sp4/Dockerfile`）。两者构建的上游版本均为 keybinder-3.0 v0.3.2，不影响本次 `check_build` 门禁；但 YAML 解析时后出现的键会覆盖前者，导致该 tag 的 `path` 映射存在歧义，后续如需按 tag 精确定位构建路径建议人工消除重复键。
- 目录名 `0.2.0/` 与其内部构建的 keybinder 版本 `0.3.2` 不再一致，属本 PR 自动升级误命名遗留；受「不得新增/重命名文件」约束未做目录改名。
- 未验证 patch/configure 阶段（本次构建失败发生在下载阶段，修正版本后与既有可用的 `0.3.2/24.03-lts-sp4` 使用同一 patch 与构建流程）。
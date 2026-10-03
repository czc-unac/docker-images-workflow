# 修复摘要

## 修复的问题
自动升级 PR #4850 使用了 GNU 镜像站并不存在的 glibc 开发期快照版本号 `2.42.9000`，导致 `wget .../glibc-2.42.9000.tar.xz` 返回 404、镜像构建失败。修复为改用它对应的正式发布版本 `2.43`。本轮核对确认 fix 分支（提交 `2f312eb53`）已包含该修复，且该修复已被实际 CI 证明有效，无需再新增代码改动。

## 修改的文件
- `Others/glibc/2.42.9000/24.03-lts-sp4/Dockerfile`: `ARG VERSION=2.42.9000` → `ARG VERSION=2.43`（下载 URL 仍为清华镜像站 `.../gnu/glibc/glibc-${VERSION}.tar.xz`）
- `Others/glibc/meta.yml`: 镜像条目 `2.42.9000-oe2403sp4` → `2.43-oe2403sp4`（path 保持 `2.42.9000/24.03-lts-sp4/Dockerfile`）
- `Others/glibc/README.md`: 标签/描述 `2.42.9000-oe2403sp4` / `glibc 2.42.9000` → `2.43-oe2403sp4` / `glibc 2.43`
- `Others/glibc/doc/image-info.yml`: 同步标签/描述为 `2.43-oe2403sp4` / `glibc 2.43`

## 修复逻辑
分析报告中定位到候选A（源码包 404），本轮直接验证并确认其为唯一真实根因，候选B/C 排除：

1. **候选A（确认）**：实测 `https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-2.42.9000.tar.xz`、`https://ftp.gnu.org/gnu/glibc/glibc-2.42.9000.tar.xz` 以及 kernel/aliyun/huaweicloud 等镜像站全部返回 **HTTP 404**；而 `glibc-2.43.tar.xz` 返回 **HTTP 200**。`.9000` 是 glibc 开发分支版本号（master 上 `2.42.9000` 即 2.43 的开发版，`2.43.9000` 才对应 2.44），GNU 只发布正式版本，故选 `2.43`（`2.42` 已存在且 tag 会与现有 `2.42-oe2403sp4` 冲突，不可用）。
2. **候选B（排除）**：在 openEuler 24.03-lts-sp4 基础镜像上按该 Dockerfile 完整执行 `configure` + `make` + `make install` 全流程构建成功，仅需已有的 `bison gcc gcc-c++ make wget xz`，未出现 `These critical programs are missing or too old`。产物镜像执行 `./bin/ldd --version` 输出 `ldd (GNU libc) 2.43`。
3. **候选C（排除）**：CI 中 `check_package_license` 为 **WARNING**（“缺少项目级Copyright声明文件”，属仓库级告警），非失败项；且现有 glibc 目录文件同样无版权头。
4. **CI 规范校验**：用上游 `eulerpublisher/update/container/app/format.py` 的 `parse_meta_yml`/`check_report` 对本次 4 个改动文件模拟，得到 `tag=2.43-oe2403sp4`、`fail_count=0`（路径/文档规范校验通过）。

真实 CI 证据（修复 PR #4873）：
- `check_build` x86_64：**SUCCESS**（#4987/#4988）。
- `check_build` aarch64：同一提交在门禁 #5216 中为 **SUCCESS**（Jenkins #5084，日志显示成功构建 `glibc-2.43` 并推送 `openeulertest/glibc:2.43-oe2403sp4-aarch64`、`Finished: SUCCESS`）；在门禁 #5217 中 aarch64 为 **FAILURE**（Jenkins #5083），但失败原因为
  `FileNotFoundError: [Errno 2] No such file or directory: 'eulerpublisher'`
  （`subprocess.call(["eulerpublisher", ...])` 找不到可执行文件），属于 CI 基础设施/工具环境问题，与本次代码改动无关。x86_64/aarch64 在环境正常时均构建成功，证明版本修复正确。

结论：原 CI 失败已由 fix 分支上的 `2.42.9000 → 2.43` 修复解决；本轮 aarch64 失败为 infra-error，**无需修改代码**（强行改动会违反最小化原则）。

## 潜在风险
- 版本目录名仍为 `2.42.9000`（受“只允许修改 PR 已涉及文件、不得新增/重命名文件”约束无法改为 `2.43`），导致元数据 tag `2.43-oe2403sp4` 与目录名不一致；仅影响可读性，`format.py` 不比较二者，不影响构建与门禁。
- `check_package_license` 的仓库级 Copyright 告警为既有 WARNING，非阻塞项，不在本 PR 修复范围。
- 若后续门禁再次出现 aarch64 失败，应优先核查 `eulerpublisher` 可执行文件是否在 `PATH` 中（infra），而非本次 glibc 代码。
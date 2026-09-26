# 修复摘要

## 修复的问题
oceanbase 5.0.1 镜像构建时因 GitHub Release tag 使用了不存在的 `v5.0.1`（实际为 `v5.0.1_CE`）导致下载 RPM 返回 404。

## 修改的文件
- `Database/oceanbase/5.0.1/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=5.0.1` 改为 `ARG VERSION=5.0.1_CE`。

## 修复逻辑
分析报告指出构建在 `[3/4]` 步从 `.../releases/download/v5.0.1/${CE_RPM}` 下载 `oceanbase-ce` RPM 时返回 404。经实际请求 GitHub API 验证：

- `https://api.github.com/repos/oceanbase/oceanbase/releases/tags/v5.0.1` 返回 `404 Not Found`，tag 不存在。
- 上游真实 tag 为 `v5.0.1_CE`（`.../tags/v5.0.1_CE` 返回 200）。
- 该 release 资产列表包含 `oceanbase-ce-5.0.1.0-100000042026072912.el8.x86_64.rpm`、`oceanbase-ce-libs-5.0.1.0-100000042026072912.el8.x86_64.rpm` 及对应 `aarch64` 资产。
- 用与 Dockerfile 完全一致的 grep 正则（含 `.el8.` 与 `${ARCH}` 拼接）对实际资产 JSON 匹配：`x86_64` 与 `aarch64` 两种架构下 `CE_RPM`、`LIBS_RPM` 均非空。
- 对最终构造的 `https://github.com/oceanbase/oceanbase/releases/download/v5.0.1_CE/${CE_RPM}` 与 `.../${LIBS_RPM}` 实际请求，`x86_64`/`aarch64` 均返回 HTTP 200。

因此仅需将 `VERSION` 修正为上游实际 tag 名 `5.0.1_CE`，即可让 `API_URL` 与 `DOWNLOAD_URL` 同时生效，最小化修复根因。其余 README.md / image-info.yml / meta.yml 仅做文档展示，与构建失败无关，未改动。

## 潜在风险
无。该改动仅影响 5.0.1 版本 Dockerfile 的下载 URL 拼接；目录名与 meta.yml key 仍为 `5.0.1`，仅作为本仓内部标识，不影响构建逻辑。`el8` 后缀与 `x86_64`/`aarch64` 资产均已验证存在于 `v5.0.1_CE`。
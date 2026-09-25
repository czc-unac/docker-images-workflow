# 修复摘要

## 修复的问题
Milvus 3.0.2 镜像构建时 MinIO 下载地址 `https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio` 返回 HTTP 410 Gone 导致构建失败，已改用仍可用的固定版本 GitHub Release 下载地址。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`: 第 41-44 行 minio 下载步骤，新增 `ARG MINIO_RELEASE=RELEASE.2025-09-07T16-13-09Z`，并将下载 URL 从 `dl.min.io` 浮动路径改为 GitHub Release 版本化资产路径。

## 修复逻辑
- 分析报告根因：`dl.min.io` 已下线 server 二进制分发，原浮动路径返回 410，`curl -fSL` 因非 2xx 退出（exit code 22）。经实测确认 `dl.min.io/server/...`（含 `archive/` 路径）对 amd64/arm64 均返回 410，且 `minio/minio` 最新 release（2025-10-15）已不再附带二进制资产。
- 修复方式：使用 GitHub Release 中仍保留二进制的最近稳定版本 `RELEASE.2025-09-07T16-13-09Z`，其资产命名规律为 `minio.linux-<arch>.RELEASE.<日期>`，可同时覆盖 `TARGETARCH=amd64` 与 `arm64`。
- 该方案与同一 Dockerfile 中已成功执行的 etcd 下载（同样走 `github.com/.../releases/download/...`）保持一致，规避了 `dl.min.io` 再次下线的风险，同时未改变后续 `chmod +x`、`mv` 逻辑。
- URL 验证：已实际请求两个架构地址，`linux-amd64` 与 `linux-arm64` 的 HEAD 均返回 HTTP 200，并对 amd64 做了 Range 下载（返回 206，内容确为 ELF 64-bit x86-64 可执行文件），确认地址有效。不涉及对第三方源文件的正则 patch。

## 潜在风险
- 固定 release 版本后续如需升级需手工调整 `MINIO_RELEASE`，但避免了浮动路径不可控下线问题。
- MinIO 2025-10-15 之后的版本不再提供 GitHub 二进制资产，本次选用其之前最后一个带资产的版本以保证可构建。
- 仓库内其他 Milvus Dockerfile（2.6.0、2.5.14）仍使用旧的 `dl.min.io` 地址，但不在本 PR 变更文件范围内，未做修改。
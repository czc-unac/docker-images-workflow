# 修复摘要

## 修复的问题
修复 `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile` 中 MinIO 二进制下载地址返回 HTTP 410 Gone 导致 Docker 构建失败的问题。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`:
  - 在运行时（stage-1）阶段新增 `ARG MINIO_VERSION=RELEASE.2025-09-07T16-13-09Z`，将 MinIO 版本固定。
  - 将 MinIO 下载地址由已下架的滚动路径 `https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio` 替换为 GitHub Releases 的版本化地址 `https://github.com/minio/minio/releases/download/${MINIO_VERSION}/minio.linux-${TARGETARCH}.${MINIO_VERSION}`。

## 修复逻辑
- 根因：`dl.min.io` 上开源版 MinIO 服务端二进制已被上游整体下架（访问任意路径均返回 410 Gone，官方页面声明社区版本文件不再提供），因此 curl -fSL 因非 2xx 以 exit code 22 中止。分析报告中"方向 1（改用版本化归档地址）"已不可行，因为 `dl.min.io` 的 `archive/` 路径同样返回 410。
- 采用分析报告"方向 2"思路：改用仍可访问的稳定分发渠道并固定版本。选择 MinIO 官方 GitHub Releases（与本 Dockerfile 第 36 行下载 etcd 的方式一致，CI 已依赖 GitHub Releases，网络可达性有保障）。
- 已验证（提交前实测）：
  - `dl.min.io` 原地址与 `archive/` 地址均返回 410（确认根因）。
  - 最新 release tag `RELEASE.2025-10-15T17-29-55Z` 未提供 linux 二进制资产（404），故不选用。
  - 选用 `RELEASE.2025-09-07T16-13-09Z`：`linux-amd64` 与 `linux-arm64` 两个架构的资产 URL 均返回 200，并实际 range 下载校验为 ELF 二进制（`curl -fSL -r 0-15` → `ELF 64-bit LSB`），满足 README 声明的 amd64/arm64 支持。
- 该修复未涉及正则 patch 第三方源文件。

## 潜在风险
- 依赖 GitHub Releases 资产命名格式 `minio.linux-<arch>.RELEASE.<tag>`；该格式为 MinIO 官方 release 资产的固定命名方式，若上游后续改变命名规则需同步调整。
- 该版本为固定 tag，不会自动获取安全更新；后续升级需手动更新 `MINIO_VERSION`。
- 同仓库 `Database/milvus/2.5.14/...`、`Database/milvus/2.6.0/...` 两个 Dockerfile 存在同样的 `dl.min.io` 下载指令，但不在本 PR 的 `changed_files` 允许范围内，按最小化原则本次未修改，建议后续单独 PR 修复。
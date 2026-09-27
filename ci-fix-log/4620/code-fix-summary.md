# 修复摘要

## 修复的问题
修复 oceanbase 5.0.1 镜像构建时因上游 GitHub release tag 名称错误（`v5.0.1` 不存在）导致下载 RPM 返回 404 的构建失败。

## 修改的文件
- `Database/oceanbase/5.0.1/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=5.0.1` 改为 `ARG VERSION=5.0.1_CE`。

## 修复逻辑
CI 失败定位在 `Database/oceanbase/5.0.1/24.03-lts-sp4/Dockerfile:15` 的下载 RUN 步骤，`curl: (22) ... 404`。

经实际请求 GitHub API 验证：
- `https://api.github.com/repos/oceanbase/oceanbase/releases/tags/v5.0.1` 返回 `{"message":"Not Found","status":"404"}`，即 tag `v5.0.1` 不存在。
- 该仓库实际 release tag 为 `v5.0.1_CE`。

Dockerfile 中 `API_URL` 与 `DOWNLOAD_URL` 均由 `v${VERSION}` 拼接，`VERSION` 错误使 API 查询 404，`grep -oP` 提取到空的 `CE_RPM`/`LIBS_RPM`，进而拼出裸路径下载 URL 并 404。将 `VERSION` 改为 `5.0.1_CE` 后，API 与下载 URL 同时被修正，属最小化单点修复，与历史 4.4.2 Dockerfile（`VERSION=4.4.2_CE_BP3` 对应 tag `v4.4.2_CE_BP3`）的约定一致。

### 正则/外部源文件验证结果
已从上游仓库实际获取并验证：
- 以 `VERSION=5.0.1_CE` 请求 `https://api.github.com/repos/oceanbase/oceanbase/releases/tags/v5.0.1_CE`，获取 asset 列表。
- 现有 `grep -oP` 正则（未改动）在 x86_64 与 aarch64 下均匹配成功：
  - `CE_RPM=oceanbase-ce-5.0.1.0-100000042026072912.el8.x86_64.rpm`
  - `LIBS_RPM=oceanbase-ce-libs-5.0.1.0-100000042026072912.el8.x86_64.rpm`
  - aarch64 对应 `.el8.aarch64.rpm`，同样匹配成功。
- 拼出的下载 URL 实测 HTTP 状态码均为 200（x86_64/aarch64 的 CE 与 libs 四个 URL 全部 200，非 404）。

综上，正则无需修改，仅修正版本 tag 即可使下载返回 2xx。

## 潜在风险
无。仅修正 Dockerfile 内部构建变量 `VERSION` 的上游 tag 名称，未改变镜像对外 tag（README/meta.yml 中的 `5.0.1-oe2403sp4`）、目录结构或下载逻辑；x86_64 与 aarch64 双架构下载 URL 均已实测返回 200。
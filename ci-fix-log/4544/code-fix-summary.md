# 修复摘要

## 修复的问题
Milvus 3.0.2 镜像构建时从已下线的 `dl.min.io` 下载 MinIO 二进制返回 HTTP 410，导致 `docker build` 在 `[stage-1 4/9]` 失败；改为在 builder 阶段用 Go 源码编译固定版本的 MinIO，并将二进制 `COPY` 进最终镜像。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`:
  - 在 builder 阶段（已安装 Go 1.24.2，第 22-26 行的 milvus 构建之后）新增一行 `RUN`，通过 `go install` 从未删除的 MinIO GitHub 源码编译并固化版本 `RELEASE.2025-09-07T16-13-09Z`，输出到 `/usr/local/bin/minio`。
  - 将最终镜像中失败的第 41-43 行 `curl ... https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio` 替换为 `COPY --from=builder /usr/local/bin/minio /usr/bin/minio`。

## 修复逻辑
1. 直接根因：分析报告中定位的 `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile:41` 使用 `dl.min.io` 的浮动 latest 直链。经实测，`dl.min.io` 现在对**所有** MinIO 社区版二进制（含 `archive/` 版本化路径、`mc`、`sha256sum`、目录列表）统一返回 **410 Gone**，页面明确声明社区版 Projetcs 已归档、不再从该站点分发任何社区版/热修复构建。因此分析报告的“方向 1（换版本化归档 URL）”实际不可行。
2. 备选渠道核验（均不可用）：
   - 官方 Docker 镜像 `minio/minio` 已从 Docker Hub 删除（`hub.docker.com/r/minio/minio` 返回 404，namespace 列表已无 `minio` 仓库），`quay.io/minio/minio`、`ghcr.io/minio/minio` 等也无可用 manifest。分析报告“方向 2（多阶段 COPY 官方镜像）”无法直接落地。
   - 主流公共镜像站（nju/bfsu/cernet/sjtug/zju/pku/aliyun/tuna/ustc 等）均无 MinIO 二进制；华为云镜像站的 `/minio/...` 路径只返回 HTML 门户页（非二进制），不可用。
   - openEuler 官方源（24.03-LTS-SP4 的 OS 与 EPOL）经 repodata 核验，只有 `python3-minio` 客户端库，没有 MinIO 服务端 RPM。
3. 最终采用“从源码构建”方案（与分析报告方向 2 中“构建绕过失效分发渠道”的思路一致）：MinIO 官方文档明确社区版现仅以源码分发，推荐 `go install github.com/minio/minio@<release>`。该方案不依赖任何第三方二进制源或外部镜像，最稳定、可复现。
4. 版本选择与工具链匹配：Dockerfile 中 `GOLANG_VERSION=1.24.2`。所选 `RELEASE.2025-09-07T16-13-09Z` 的 `go.mod` 为 `go 1.24.0 / toolchain go1.24.2`，与本镜像 Go 版本完全一致；并显式设置 `GOTOOLCHAIN=local`，确保构建期间不会触发额外 toolchain 下载。`CGO_ENABLED=0` 生成静态链接二进制，`-ldflags` 做了 `-s -w` 裁剪并写入 Version/ReleaseTag，既减小体积（约 151MB→111MB）又能正确输出版本号。
5. 复用现有的 builder 阶段（该阶段已具备 Go 与 git 环境），无需新增基础镜像，最终镜像体积与依赖不变。

### 实际验证（提交前完成）
- 已用与 Dockerfile 完全一致的 **Go 1.24.2** 工具链执行本次要写入的正则/命令：
  `CGO_ENABLED=0 GOTOOLCHAIN=local GOBIN=... go install -ldflags="-s -w -X .../cmd.Version=RELEASE.2025-09-07T16-13-09Z -X .../cmd.ReleaseTag=RELEASE.2025-09-07T16-13-09Z" github.com/minio/minio@RELEASE.2025-09-07T16-13-09Z`
  结果 `exit=0`，生成 ELF 64-bit LSB **statically linked / stripped** 二进制，`minio --version` 正确输出 `RELEASE.2025-09-07T16-13-09Z`（Runtime: go1.24.2）。
- 已确认 `go install` 能在 `GOTOOLCHAIN=local` 下完成，不需要下载 go1.24.8 等工具链。
- 已用 BuildKit 对修改后的 Dockerfile 执行静态检查（`docker build --check`）：`Check complete, no warnings found.`，且 `openeuler/openeuler:24.03-lts-sp4` 基础镜像元数据可正常解析。
- 说明：本修复不是对第三方源文件做正则 patch，而是替换下载来源；上述为实际构建/下载可用性验证。

## 潜在风险
- 构建耗时增加：builder 阶段需拉取并编译 MinIO 全部 Go 依赖（几分钟量级），CI 时间会变长；但产物可复现、无外部失效链接风险。
- 架构适配：`go install` 在 buildx 各目标平台的原生 builder 中执行，无需 `GOARCH`，amd64/arm64 均按目标架构编译；已在本机 amd64 验证，arm64 依赖 Go 原生交叉/原生支持，未单独实机验证 arm64 二进制。
- 同一 `dl.min.io` 失效问题也存在于 `Database/milvus/2.5.14/24.03-lts-sp2/Dockerfile` 与 `Database/milvus/2.6.0/24.03-lts-sp2/Dockerfile`，但不在本 PR 的 `changed_files` 范围内，未作修改；若 CI 本轮回建这两个版本，仍会因同一原因失败。
- README.md / doc/image-info.yml 中示例输出的 MinIO 版本号（`RELEASE.2025-06-13T11-33-47Z`）与新固定版本不完全一致，属文档示例，与 CI 失败无直接关系，按最小化原则未改动。
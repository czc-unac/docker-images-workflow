# 修复摘要

## 修复的问题
修复 Milvus 3.0.2 Dockerfile 在 openEuler 24.03-LTS-SP4 上构建失败：MinIO 官方二进制下载地址已返回 410 Gone，且上游 `install_deps.sh` 不识别 openEuler 发行版。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`:
  - 新增 `minio` 构建阶段：在 `golang:1.24-alpine` 中以 `CGO_ENABLED=0` 从源码编译 MinIO（固定到已归档的 release commit）。
  - 将运行阶段的 `RUN curl ... dl.min.io ... /minio` 替换为 `COPY --from=minio /go/bin/minio /usr/bin/minio`。
  - 在 `make` 前用 `sed` 为上游 `scripts/install_deps.sh` 的发行版分支增加 `openEuler|openeuler`，使其复用 dnf 分支安装依赖。

## 修复逻辑
提供的分析报告因上下文日志缺失而判定“证据不足”。为定位真实根因，直接拉取了该 PR 门禁结果中失败构建 job 的 Jenkins 控制台日志：

- x86_64: `https://ci.openeuler.openatom.cn/job/multiarch/job/openeuler/job/x86-64/job/openeuler-docker-images/5015/consoleText`
- aarch64: `https://ci.openeuler.openatom.cn/job/multiarch/job/openeuler/job/aarch64/job/openeuler-docker-images/5111/consoleText`

两份日志的失败点一致（exit code: 22）：

```
#11 [stage-1 4/9] RUN curl -fSL -o minio https://dl.min.io/server/minio/release/linux-amd64/minio ...
curl: (22) The requested URL returned error: 410
```

根因与修复：

1. **MinIO 二进制已停止分发（HTTP 410 Gone）**：`dl.min.io` 对所有社区版本返回 410，正文明确说明 “The open-source MinIO Server ... projects are archived and no longer maintained. These files are no longer served from this site.”；MinIO 官方 README 亦声明社区版改为仅源码分发。经核验 `dl.min.io`、`repo.huaweicloud.com`、各高校镜像均无可用二进制，且 `minio/minio` / `quay.io/minio/minio` 镜像已不可匿名拉取。因此改为按官方推荐方式从源码编译：`golang:1.24-alpine` + `CGO_ENABLED=0` + `go install github.com/minio/minio@<release commit>`，得到静态链接二进制后 `COPY` 进运行阶段。该方式已在本地验证：minio 阶段构建成功，产物 `/go/bin/minio` 可执行（`minio version ... Runtime: go1.24.13 linux/amd64`）。所用 commit `9e49d5e7a648f00e26f2246f4dc28e6b07f8c84a` 对应上游最后一个 release tag `RELEASE.2025-10-15T17-29-55Z`。

2. **上游 `install_deps.sh` 不识别 openEuler**：Milvus 3.0.2 的 `scripts/install_deps.sh` 通过 `/etc/os-release` 的 `ID` 做发行版分支判断，openEuler 的 `ID="openEuler"` 不匹配 `ubuntu|debian`、`rocky|almalinux`、`amzn`、`centos|rhel`，落入 `*)` 分支 `exit 1`（本地以 openEuler 基础镜像实测报错 `Unsupported Linux distribution: openEuler`）。该构建阶段仅被 MinIO 的并发失败提前取消，修复 MinIO 后必然触发。修复方式为在 `git clone` 后、调用脚本前用 `sed` 将 `amzn)` 分支扩展为 `amzn|openEuler|openeuler)`，复用其 dnf 分支。
   - 正则验证：已从上游 `milvus-io/milvus` 的 `v3.0.2` tag 获取 `scripts/install_deps.sh` 验证，`re.sub`/`replace` 对 `amzn)` 精确匹配 1 处，替换结果正确（`amzn|openEuler|openeuler) install_amazon_linux_deps`）。
   - 该分支的系统包列表 `wget curl which git make ninja-build gcc gcc-c++ gcc-gfortran automake python3-devel python3-pip libaio libuuid-devel zip unzip ccache libtool m4 autoconf openssl-devel zlib-devel` 已在 openEuler 24.03-LTS-SP4 容器中实测可完整安装（exit 0），随后的 cmake/conan/rust 安装与发行版无关。

3. **版权检查并非失败项**：真实门禁表格中 `check_package_license` 为 `⚠ WARNING`（“缺少项目级Copyright声明文件”），真正失败的是 `x86_64/aarch64 check_build`，因此未做版权头改动，避免无关修改。

最终以 `docker build --check` 校验该 Dockerfile：`Check complete, no warnings found.`

## 潜在风险
- 运行阶段改为 `COPY --from=minio`，需要 CI 构建环境可访问 Docker Hub 基础镜像 `golang:1.24-alpine`（原 Dockerfile 已依赖 Docker Hub 的 `openeuler/openeuler`，网络可达性一致）。该 minio 阶段已本地构建验证。
- MinIO 开源项目已归档，固定 commit 的源码仍可从 GitHub/Go module proxy 获取；若未来上游仓库下线，需改用其他对象存储或镜像方案。
- 已额外修复 `install_deps.sh` 识别 openEuler 的问题，属于同一构建链条上的后续阻塞点；未改动的 `make build-cpp` / `make build-go` 仍可能因上游要求（如 Go 1.26.6 由 GOTOOLCHAIN 自动获取）在个别网络环境下失败，但本次已完成可静态验证的两个确定阻塞点的修复。
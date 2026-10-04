# 修复摘要

## 修复的问题
修复 milvus 3.0.2 镜像构建时 MinIO 二进制下载返回 HTTP 410（Gone）导致的 Docker 构建失败（amd64/arm64 双架构均失败）。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`: 将 MinIO 下载地址从 `https://mirrors.huaweicloud.com/minio/server/minio/release/linux-$TARGETARCH/archive/minio.RELEASE.2025-10-15T17-29-55Z/minio` 改为 GitHub Release 官方制品 `https://github.com/minio/minio/releases/download/RELEASE.2025-09-07T16-13-09Z/minio.linux-$TARGETARCH.RELEASE.2025-09-07T16-13-09Z`。

## 修复逻辑
分析报告未提供 CI 日志（`infra-error / 证据不足`），因此本次通过 GitCode API 定位到 PR #4851 门禁评论中的真实构建 job，并直接拉取了两架构的完整控制台日志：

- x86_64: `log-ci.openeuler.openatom.cn/job/multiarch/openeuler/x86-64/openeuler-docker-images/4965`
- aarch64: `log-ci.openeuler.openatom.cn/job/multiarch/openeuler/aarch64/openeuler-docker-images/5061`

两个 job 的第一条（也是唯一）error 完全一致，均发生在 `Dockerfile:41` 的 MinIO 下载步骤：

```
#10 [stage-1 4/9] RUN curl -fSL -o minio https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio ...
curl: (22) The requested URL returned error: 410
ERROR: failed to solve: ... exit code: 22
```

根因：MinIO 已停止通过 `dl.min.io/server/minio/release/...` 分发社区版二进制（返回 410 Gone），旧版本 `2.6.0` 镜像已构建完成不受影响，而新增的 `3.0.2` 需要重新下载因而失败。

修复验证：
1. 目录中当前已被上一次修复改为华为云镜像站 URL，但经验证该 URL 返回的是 `text/html` SPA 页面（HTTP 200，`content-type: text/html`，实际是 12KB HTML），并非可执行的 MinIO 二进制，`curl -fSL` 不会报错，会产出损坏的镜像，因此不可用。
2. 经 `api.github.com/repos/minio/minio/releases` 核对，官方 Release `RELEASE.2025-10-15T17-29-55Z` 不含任何二进制资产；最后一个包含 Linux amd64/arm64 二进制的 Release 为 `RELEASE.2025-09-07T16-13-09Z`。
3. 已实测两个目标 URL 均返回 `302 → 200`、`content-type: application/octet-stream`，且文件头为 ELF（`7f 45 4c 46`），并与 `$TARGETARCH`（amd64/arm64）命名精确对应，GitHub 与 etcd 下载同源（CI 中已验证可达）。

修复与知识库“模式16（改用多阶段构建/更换分发源绕过上游停止发布）”方向一致，只改动下载源，最小化改动。

## 潜在风险
- 采用固定的 MinIO Release（`RELEASE.2025-09-07T16-13-09Z`）而非 `latest` 滚动地址；后续如需升级需手动更新版本号。这是为规避上游停止发布而做的有意取舍，功能上无影响。
- 本次日志在 MinIO 步骤即失败，builder 阶段（`make build-cpp` / `make build-go`）因被取消尚未跑到；已核对 `scripts/install_deps.sh`（v3.0.2）与 Dockerfile 中的 4 条 `sed` patch 均能匹配，但完整编译成功仍需下一次 CI 验证。
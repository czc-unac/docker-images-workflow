# CI 失败分析报告

## 基本信息
- PR: #4723 — 【自动升级】milvus容器镜像升级至3.0.2版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: MinIO下载源410
- 新模式症状关键词: curl: (22), 410 Gone, dl.min.io, minio, 下载地址失效

## 日志-状态一致性前置检查
`ci.logs` 末尾为 `Finished: FAILURE`（非 `Finished: SUCCESS` / `Build successful`），且日志中包含真实的下游 Docker 构建步骤（`[builder 2/4]`、`[stage-1 4/9]`、`Dockerfile:41`），因此不适用"证据不足"判定，可正常定位根因。

## 根因分析

### 直接错误
```
#10 [stage-1 4/9] RUN curl -fSL -o minio https://dl.min.io/server/minio/release/linux-amd64/minio &&     chmod +x ./minio &&     mv ./minio /usr/bin/
#10 0.065   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
#10 0.065  0   405    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
#10 0.582 curl: (22) The requested URL returned error: 410
#10 ERROR: process "/bin/sh -c curl -fSL -o minio https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio &&     chmod +x ./minio &&     mv ./minio /usr/bin/" did not complete successfully: exit code: 22
------
Dockerfile:41
  41 | >>> RUN curl -fSL -o minio https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio && \
  42 | >>>     chmod +x ./minio && \
  43 | >>>     mv ./minio /usr/bin/
ERROR: failed to solve: process "..." did not complete successfully: exit code: 22
```

### 根因定位
- 失败位置: `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile:41-43`
- 失败原因: 从 `https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio` 下载 MinIO 二进制时，服务器返回 HTTP **410 Gone**（资源已被永久移除），`curl -f` 因此以 exit code 22 退出，`RUN` 步骤失败并导致 Docker BuildKit 整个构建中断。

### 与 PR 变更的关联
失败步骤是本次 PR **新增** Dockerfile 中的第 41-43 行（`minio` 下载），与 PR 改动直接相关，属确定性根因。同一 Dockerfile 中其它外部下载均已成功（`#9` etcd 下载 `DONE 1.3s`）、两处 `yum install` 也已正常完成，说明失败是局部、单点的下载源问题，而非编译/依赖链问题。

## 修复方向

### 方向 1（置信度: 高）
更换 MinIO 二进制的下载来源/路径。`dl.min.io/server/minio/release/` 这一旧发布路径已返回 410 Gone，需改用 MinIO 当前实际有效的官方分发路径或镜像站地址。**注意**：不能仅套用历史 404 模式，410 表示该路径已被上游永久下架，需确认真实的新路径后再替换。

### 方向 2（置信度: 中）
参考知识库**模式16（RPM 包停止发布 → 多阶段构建绕过）**：以官方 `minio/minio` 镜像作为构建源阶段，用 `COPY --from=...` 提取 `minio` 二进制到最终镜像，从而绕开对 `dl.min.io` 直链下载的依赖。此方向不改动下载 URL，但需确认官方镜像 Tag 与 `TARGETARCH` 的对应关系。

## 需要进一步确认的点
1. MinIO 当前官方二进制分发的**真实有效 URL**（`dl.min.io` 是否更名/迁移至 AIStor 路径，或是否存在其它官方发布地址），必须在提交前实测确认。
2. 上游 `milvus` 3.0.2 的部署/构建流程是否确实需要 `minio` 二进制，以及所需的最低版本。
3. aarch64 架构是否存在与 amd64 相同的 410 问题（当前日志为 amd64，需确认 arm64 的下载路径同样可达）。
4. 若采用方向 2，需确认 `minio/minio` 官方镜像是否存在可用的多架构（amd64/arm64）Tag。

## 修复验证要求
本问题的修复方向包含"替换外部下载 URL"，属于对外部源的依赖变更，code-fixer 在提交前**必须实测验证**，不能凭推测直接填写地址：
- 对拟采用的新 MinIO 下载 URL 执行 `curl -fSL`（含 amd64 与 arm64 两种 `TARGETARCH`），确认返回 HTTP 200，且下载文件为可执行的 ELF 二进制（而非 HTML/错误页）。
- 若采用多阶段构建方向，需实际拉取所选 `minio/minio` 镜像 Tag，确认 `COPY` 源路径中的二进制存在且两架构均可获取。
- 验证完成后，将确认可用的来源信息随修复一并说明。

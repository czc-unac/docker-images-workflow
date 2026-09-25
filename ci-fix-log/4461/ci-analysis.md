# CI 失败分析报告

## 基本信息
- PR: #4461 — 【自动升级】milvus容器镜像升级至3.0.2版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: MinIO下载源410
- 新模式症状关键词: curl: (22), 410, dl.min.io, minio, exit code: 22

## 根因分析

### 直接错误
```
#10 [stage-1 4/9] RUN curl -fSL -o minio https://dl.min.io/server/minio/release/linux-amd64/minio &&     chmod +x ./minio &&     mv ./minio /usr/bin/
#10 0.080   % Total    % Received % Xferd  Average Speed ...
#10 0.080  0   405    0     0    0     0 --:--:-- --:--:-- --:--:--     0
#10 0.580 curl: (22) The requested URL returned error: 410
#10 ERROR: process "/bin/sh -c curl -fSL -o minio https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio &&     chmod +x ./minio &&     mv ./minio /usr/bin/" did not complete successfully: exit code: 22
...
Dockerfile:41
  41 | >>> RUN curl -fSL -o minio https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio && \
ERROR: failed to solve: process "..." did not complete successfully: exit code: 22
```

### 根因定位
- 失败位置: `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile:41`（新增文件的 minio 下载 RUN 步骤）
- 失败原因: MinIO 二进制下载 URL `https://dl.min.io/server/minio/release/linux-<arch>/minio` 返回 HTTP **410 Gone**（资源已被上游永久移除），`curl -fSL` 因非 2xx 状态码退出（exit code 22），导致 Docker 构建在该层失败。

### 与 PR 变更的关联
该 Dockerfile 为本 PR 新增（`new_file: True`），第 41-43 行的 minio 下载指令即本 PR 引入。日志中同一 `stage-1` 阶段的前一步 etcd 下载（#9）已成功，唯一失败步骤就是新加入的 minio 下载（#10）。因此失败**直接由本 PR 新增内容触发**，与历史存量代码无关。

注：日志末尾为 `Build step 'Execute shell' marked build as failure` + `Finished: FAILURE`，非 `Finished: SUCCESS`，前置状态一致性检查通过，日志确为该构建 job 的真实失败日志。

## 修复方向

### 方向 1（置信度: 高）
将 minio 下载源改为当前仍可用的地址，再执行 `chmod +x` 与 `mv`。可选做法（不提供代码）：
- 改从 MinIO 官方版本化归档路径下载固定 release（如 `.../archive/minio.RELEASE.<日期>`），避免使用已被 410 移除的 `release/linux-<arch>/minio` 浮动路径；
- 或改用其他可达镜像/分发渠道（如官方 GitHub release 资产或国内镜像站）提供的 minio 二进制。
建议在提交前手工验证目标 URL 返回 200。

### 方向 2（置信度: 中）
若 `dl.min.io` 仅是路径/命名调整，可保留域名但修正完整路径（例如加入 release 版本号或 `archive/` 前缀）。需先确认 MinIO 当前对外发布的实际路径格式，否则仍可能失败。

## 需要进一步确认的点
- 确认 MinIO 官方当前对外提供的二进制下载入口及是否存在可直接下载 `minio` 二进制的稳定地址（`dl.min.io` 返回 410 说明原路径已被移除）。
- 确认 `TARGETARCH`（amd64/arm64）在多架构构建下均有对应的可下载制品。
- 确认 minio 下载步骤是否应与 etcd 一样使用带 `-fSL` 的 curl，并考虑对下载源可用性做兜底（如备用 URL），以降低后续再次因上游下线而失败的风险。

## 修复验证要求
不涉及对第三方源文件的正则 patch，无需额外上游源码校验。但 Code Fixer 提交前必须实际请求所选的新 minio 下载 URL，确认返回 HTTP 200（而非 3xx/4xx），并保证 `linux-amd64` 与 `linux-arm64` 两个架构的地址均有效。

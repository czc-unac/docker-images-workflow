# CI 失败分析报告

## 基本信息
- PR: #4626 — 【自动升级】milvus容器镜像升级至3.0.2版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: MinIO下载源410
- 新模式症状关键词: curl: (22), 410 Gone, dl.min.io, minio, release/linux-amd64

## 根因分析

### 直接错误
```
#12 [stage-1 4/9] RUN curl -fSL -o minio https://dl.min.io/server/minio/release/linux-amd64/minio &&     chmod +x ./minio &&     mv ./minio /usr/bin/
#12 0.057   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
#12 0.057                                  Dload  Upload   Total   Spent    Left  Speed
#12 0.057 \r  0     0    0     0    0     0      0 --:--:-- --:--:-- --:--:--     0\r  0   405    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
#12 0.616 curl: (22) The requested URL returned error: 410
#12 ERROR: process "/bin/sh -c curl -fSL -o minio https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio &&     chmod +x ./minio &&     mv ./minio /usr/bin/" did not complete successfully: exit code: 22
...
Dockerfile:41
  41 | >>> RUN curl -fSL -o minio https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio && \
  42 | >>>     chmod +x ./minio && \
  43 | >>>     mv ./minio /usr/bin/
ERROR: failed to solve: process "..." did not complete successfully: exit code: 22
```
日志末尾为 `Finished: FAILURE`（构建确已失败，非成功日志掩盖问题）。首个致命 error 即上方的 `curl: (22) ... error: 410`，后续 `Build step 'Execute shell' marked build as failure` 皆为该错误的级联结果。

### 根因定位
- 失败位置: `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile:41`
- 失败原因: Dockerfile 运行时阶段（stage-1）用 `curl` 从 `https://dl.min.io/server/minio/release/linux-${TARGETARCH}/minio` 下载 MinIO 服务端二进制，该 URL 返回 HTTP **410 Gone**（资源已永久下架），`curl -fSL` 因非 2xx 状态码以 exit code 22 失败，整个 Docker build 中止。

### 与 PR 变更的关联
本 PR 新增了 `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`（new_file），其中第 41 行新增了这条 MinIO 下载指令。失败正是由该新增步骤直接触发，与 PR 变更**直接相关**，非历史遗留问题。

补充说明：日志中另一处 `#7 108.8 [MIRROR] llvm-libs-...: Curl error (92): Stream error in the HTTP/2 framing layer ...` 为镜像站瞬时网络抖动，yum 随即重试并成功安装（`#7 127.3 Installing : ... 17/17`、`#7 128.9 Complete!`），**不是**本次根因，不应据此判断。

## 修复方向

### 方向 1（置信度: 高）
`dl.min.io` 的“无版本号”下载路径 `/server/minio/release/linux-${TARGETARCH}/minio` 已被上游下架并返回 410。应改用仍然可用的 **版本化归档地址**（例如 `dl.min.io/server/minio/release/linux-${TARGETARCH}/archive/minio.RELEASE.<具体时间戳>`）将 MinIO 固定到一个仍可下载的稳定 release tag，避免依赖会被移除的滚动路径。

### 方向 2（置信度: 中）
若上游下载策略整体变更导致原域名不可用，可改用其他可靠的 MinIO 二进制分发渠道/镜像站（或从官方容器镜像中多阶段 `COPY` 二进制，类似知识库模式16的做法），并在 Dockerfile 中固定具体版本以保证可复现。

### 方向 3（置信度: 低）
仅对 `curl` 增加 `--retry` 无法解决 410（410 为永久性下架，重试无效），不作为独立修复方案，仅可作为其他方案的健壮性补充。

## 需要进一步确认的点
1. 需确认 MinIO 官方当前实际可用的版本化下载 URL 形态（RELEASE tag 命名与 `archive/` 路径前缀），再据此替换，避免再次 404/410。
2. 需确认该 URL 在 **amd64 与 arm64** 两个架构下均存在对应二进制（Dockerfile 使用 `$TARGETARCH`，README 声明支持 amd64/arm64）。
3. 若采用方式“从官方镜像 COPY 二进制”，需确认对应镜像的 tag 与二进制路径。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
不涉及正则 patch 第三方源文件。但 **code-fixer 在提交前必须实际验证替换后的 MinIO 下载 URL 可访问**：分别以 `linux-amd64` 与 `linux-arm64` 构造 URL 并执行 `curl -fSL -o /dev/null`（或 HEAD 请求），确认返回 2xx 而非 410/404，验证通过后再提交。由于置信度较高但上游 URL 形态需现场确认，禁止假设任意 URL 一定可用。

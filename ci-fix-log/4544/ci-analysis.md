# CI 失败分析报告

## 基本信息
- PR: #4544 — 【自动升级】milvus容器镜像升级至3.0.2版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: minio下载源410
- 新模式症状关键词: curl: (22), 410, dl.min.io, minio, release/linux-amd64

> 前置检查：日志末尾为 `Finished: FAILURE`，不存在 `Finished: SUCCESS` / `Build successful` 成功标志，日志与 PR 失败状态一致，正常进入分析。

## 根因分析

### 直接错误
```
#10 [stage-1 4/9] RUN curl -fSL -o minio https://dl.min.io/server/minio/release/linux-amd64/minio &&     chmod +x ./minio &&     mv ./minio /usr/bin/
#10 0.081   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
#10 0.082 \r  0   405    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
#10 0.582 curl: (22) The requested URL returned error: 410
#10 ERROR: process "/bin/sh -c curl -fSL -o minio https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio &&     chmod +x ./minio &&     mv ./minio /usr/bin/" did not complete successfully: exit code: 22
------
Dockerfile:41
  40 |
  41 | >>> RUN curl -fSL -o minio https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio && \
  42 | >>>     chmod +x ./minio && \
  43 | >>>     mv ./minio /usr/bin/
  44 |
------
ERROR: failed to solve: process "/bin/sh -c curl -fSL -o minio ... " did not complete successfully: exit code: 22
Finished: FAILURE
```

### 根因定位
- 失败位置: `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile:41`
- 失败原因: 下载 MinIO 二进制时，URL `https://dl.min.io/server/minio/release/linux-amd64/minio` 返回 HTTP **410 Gone**（服务端仅返回 405 字节错误页），`curl -fSL` 因此以 exit code 22 退出，Docker 构建在 `[stage-1 4/9]` 步骤失败，整个 `docker build` 中止。

补充说明（排除干扰项）：
- 日志中 `#7 [MIRROR] perl-Math-BigInt ... Curl error (56): Failure when receiving data from the peer` 属于 yum 镜像源的瞬时网络抖动，随后已重试成功（`#7 68.98 (153/207): perl-Math-BigInt ... 260 kB/s | 136 kB`），**不是**本次失败根因。
- 同一构建中 `#9` 的 etcd 下载（同样使用 `$TARGETARCH` 直链）成功下载 18.4M 并 `DONE 1.1s`，说明架构变量与网络本身正常，唯独 minio 地址失效。

### 与 PR 变更的关联
直接由本 PR 引入。PR 新增 `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`（新文件），其中第 41 行即为失败的 minio 下载指令；该 Dockerfile 同时通过 `meta.yml` 新增 `3.0.2-oe2403sp4` 条目被 CI 纳入构建。因此这是本次 PR 改动直接触发的失败，与历史无关。

## 修复方向

### 方向 1（置信度: 高）
将 minio 的下载地址从已下线的“latest 直链”改为 MinIO 当前实际可用的**版本化归档地址**，并显式固定版本号（例如形如 `.../server/minio/release/linux-${TARGETARCH}/archive/minio.<RELEASE.日期T时间Z>`），避免继续依赖会被下线的浮动路径。需先从 MinIO 官方确认目标版本归档 URL 真实存在后再提交。

### 方向 2（可选，置信度: 中）
若 MinIO 官方已不再以二进制直链形式提供该版本，则改用替代分发渠道（如稳定的第三方镜像源），或改为多阶段构建 `COPY --from=<官方 minio 镜像> /usr/bin/minio /usr/bin/`（参考模式16“RPM 包停止发布→多阶段构建绕过”的思路），从官方镜像中提取二进制。

## 需要进一步确认的点
1. `https://dl.min.io/server/minio/release/linux-amd64/minio` 返回 410 的确切原因——是 MinIO 下线了社区版 latest 直链，还是迁移到 `.../archive/minio.<版本>` 路径；需访问 MinIO 官方发布页/`dl.min.io` 目录确认当前正确 URL 形态。
2. 目标 `TARGETARCH`（amd64/arm64）对应的归档文件名/路径是否都可用，需 x86-64 与 aarch64 两个架构分别验证，避免修复后另一架构仍 404/410。
3. 该地址失效是否为上游长期策略变更（若是，应固定版本而非仅换 latest 链接），建议在 Dockerfile 中固化版本号。

## 修复验证要求
本修复方向为“替换下载 URL”（非正则 patch 外部源文件），无需拉取上游源码做正则匹配验证。但 code-fixer 在提交前**必须**实际访问所更换后的 MinIO 下载 URL，确认其返回 200 且能下载到非 HTML 错误页的二进制文件，并同时验证 amd64、arm64 两个架构路径均可下载，避免以 410/404 或错误页替换旧 URL。

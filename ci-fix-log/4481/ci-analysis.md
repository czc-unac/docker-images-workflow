# CI 失败分析报告

## 基本信息
- PR: #4481 — 【自动升级】flume容器镜像升级至1.10.1版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 模式01（Apache CDN 旧版本 404；与模式38 ActiveMQ dlcdn 404、模式02下载URL版本路径错误同源）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#7 [2/3] RUN curl -fSL -o flume.tar.gz https://dlcdn.apache.org/flume/1.10.1/apache-flume-1.10.1-bin.tar.gz;     mkdir -p /usr/local/flume &&     tar -zxf flume.tar.gz -C /usr/local/flume --strip-components=1 &&     rm -rf flume.tar.gz
#7 0.061 \r  0   196    0    0    0    0      0      0 --:--:-- --:--:-- --:--:--     0
#7 0.410 curl: (22) The requested URL returned error: 404
#7 0.416 tar (child): flume.tar.gz: Cannot open: No such file or directory
#7 0.416 tar: Error is not recoverable: exiting now
#7 ERROR: process "/bin/sh -c curl -fSL -o flume.tar.gz ..." did not complete successfully: exit code: 2
...
Dockerfile:6
   6 | >>> RUN curl -fSL -o flume.tar.gz https://dlcdn.apache.org/flume/${VERSION}/apache-flume-${VERSION}-bin.tar.gz; \
ERROR: failed to solve: process ... did not complete successfully: exit code: 2
```

### 根因定位
- 失败位置: `Bigdata/flume/1.10.1/24.03-lts-sp4/Dockerfile` 第 6 行（`ARG VERSION=1.10.1`）
- 失败原因: Docker 构建时从 `dlcdn.apache.org` 下载 `apache-flume-1.10.1-bin.tar.gz` 返回 HTTP 404。`dlcdn.apache.org` 是 Apache 官方 CDN 分发节点，通常只保留当前最新版本，1.10.1 已从 CDN 下架，因此该 URL 不存在；curl 因 `-f` 校验失败（exit 22）未生成文件，随后 `tar` 报文件不存在（exit 2），整个 `RUN` 步骤以 exit code 2 失败。日志中请求路径 `.../flume/1.10.1/...` 为 VERSION 变量正常展开，说明 URL 结构本身无误，纯粹是上游制品在该源不可用。

### 与 PR 变更的关联
该 PR 新增了 `Bigdata/flume/1.10.1/24.03-lts-sp4/Dockerfile`（新文件），其第 6 行正是上述下载命令。因此失败由本次 PR 新增的 Dockerfile 直接触发，与仓库既有代码无关。README、`doc/image-info.yml`、`meta.yml` 的改动仅做条目登记，非失败原因。注意 `meta.yml` 末尾缺少换行（`\ No newline at end of file`）为格式瑕疵，但并非本次构建失败根因。

## 修复方向

### 方向 1（置信度: 高）
将 flume 1.10.1 的下载源从只保留最新版的 `dlcdn.apache.org` 改为保留历史版本的 Apache 归档源（如 `archive.apache.org/dist/flume/1.10.1/`），或改用已验证可达的国内镜像站（如 `repo.huaweicloud.com` 的 apache 目录）。修复后需确认目标源确实存在 `apache-flume-1.10.1-bin.tar.gz`。
> 参照知识库模式01/模式33/模式38：Apache CDN 对旧版本 404 的通用解法为换归档源或换可信镜像站。

### 方向 2（可选）
若该镜像确无历史版本可用（1.10.1 已彻底下架），则考虑调整版本策略（例如不再新增 1.10.1，改为对齐当前 CDN 可用的 1.11.0）。此方向属于版本选择问题，需人工确认开放需求。

## 需要进一步确认的点
- 确认 `archive.apache.org/dist/flume/1.10.1/apache-flume-1.10.1-bin.tar.gz`（或拟采用镜像站对应路径）在构建网络环境下可访问且返回 gzip 制品，而非 HTML/错误页。
- 核实镜像站对 CI 构建环境的可达性（历史上有 `downloads.apache.org`、`archive.apache.org` 网络不可达的案例，见模式33）。
- `meta.yml` 文件末尾缺少换行符，建议一并核对 CI 是否对 YAML 末尾换行有校验要求（本次失败日志未体现，属附带检查项）。

## 修复验证要求
本次根因不属于"正则 patch 第三方源文件"场景，无需额外上游文件正则验证。但修复涉及替换外部下载源，code-fixer 在提交前应确认新下载源 URL 可解析到具体制品（建议人工/CI 侧验证 HTTP 返回 200 且为 gzip），避免再次落入 404 或网络不可达。

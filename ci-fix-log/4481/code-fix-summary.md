# 修复摘要

## 修复的问题
将 flume 1.10.1 镜像的下载源从只保留最新版本的 `dlcdn.apache.org` 改为 Apache 官方历史归档源 `archive.apache.org/dist`，解决构建时 HTTP 404 导致的 build-error。

## 修改的文件
- `Bigdata/flume/1.10.1/24.03-lts-sp4/Dockerfile`: 第 6 行下载 URL 由 `https://dlcdn.apache.org/flume/${VERSION}/apache-flume-${VERSION}-bin.tar.gz` 改为 `https://archive.apache.org/dist/flume/${VERSION}/apache-flume-${VERSION}-bin.tar.gz`。

## 修复逻辑
分析报告根因：`dlcdn.apache.org` 仅保留当前最新版本，历史版本 1.10.1 已被下架，`curl -fSL` 因 404（exit 22）失败，导致后续 `tar` 找不到文件（exit 2），`RUN` 步骤整体失败。修复采用报告“方向 1”，换用长期保存历史版本的 Apache 归档源。该写法与仓库既有约定一致（如 `Bigdata/storm/2.8.2/.../Dockerfile`、`Bigdata/parquet/...` 均使用 `https://archive.apache.org/dist/...` 形式），仅替换域名/路径前缀，未改动其他逻辑，改动最小。

验证结果：已实际请求 `https://archive.apache.org/dist/flume/1.10.1/apache-flume-1.10.1-bin.tar.gz`，返回 `HTTP/1.1 200 OK`，`Content-Type: application/x-gzip`，`Content-Length: 88977860`，确认为真实 gzip 制品而非 HTML 错误页，URL 可达且路径正确。

## 潜在风险
无。仅将不可用的 CDN 源替换为历史归档源，归档源稳定保存 Apache 历史发布版本，不会影响镜像功能。未修改 `1.11.0`（当前 CDN 仍可用版本）及 README/doc/meta.yml 等非根因文件。
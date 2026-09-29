# 修复摘要

## 修复的问题
修复 milvus 3.0.2 Dockerfile 构建时从 `dl.min.io` 下载 MinIO 二进制返回 HTTP 410 Gone 导致的构建失败。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`: 将第 41 行 MinIO 下载 URL 的路径由 `https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio` 改为 `https://dl.min.io/aistor/minio/release/linux-$TARGETARCH/minio`。

## 修复逻辑
CI 分析报告指出根因是 `Dockerfile:41` 从旧发布路径 `dl.min.io/server/minio/release/...` 下载 MinIO 时服务器返回 **410 Gone**（该路径被上游永久下架），`curl -f` 因此以 exit code 22 退出，导致构建中断。本次修复仅替换为 MinIO 当前实际有效的官方分发路径（AIStor 路径），不改动下载动作、目标文件名或后续 `chmod`/`mv` 逻辑，属于最小化改动。

**已实测验证**（在提交前完成）：
- `https://dl.min.io/server/minio/release/linux-amd64/minio` → HTTP 410（确认旧路径失效）。
- `https://dl.min.io/aistor/minio/release/linux-amd64/minio` → HTTP 200，`content-length: 278278306`，`content-type: application/octet-stream`。
- `https://dl.min.io/aistor/minio/release/linux-arm64/minio` → HTTP 200，`content-length: 256114850`，`content-type: application/octet-stream`。
- 两架构下载文件起始字节均为 `7f 45 4c 46`（`\x7fELF`），确认为可执行 ELF 二进制，而非 HTML/错误页。

## 潜在风险
本次仅替换 URL 路径，未改变镜像构建的其它步骤与最终镜像结构，风险很低。需要留意的是 MinIO 上游分发路径处于迁移期（`server/` → `aistor/`），若上游后续再次调整路径，该地址可能再次失效；届时需同步更新。
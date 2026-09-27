# 修复摘要

## 修复的问题
自动升级 PR 将 alluxio 版本错误地提升到上游并不存在的 2.9.6，导致下载 URL `https://downloads.alluxio.io/downloads/files/2.9.6/alluxio-2.9.6-bin.tar.gz` 返回 HTTP 404、Docker 构建失败（curl exit code 22）；现回退到存在官方二进制包的 2.9.5。

## 修改的文件
- `Storage/alluxio/2.9.6/24.03-lts-sp4/Dockerfile`: `ARG VERSION="2.9.6"` 改回 `ARG VERSION="2.9.5"`
- `Storage/alluxio/meta.yml`: 条目键 `2.9.6-oe2403sp4` 改回 `2.9.5-oe2403sp4`（path 仍为 `2.9.6/24.03-lts-sp4/Dockerfile`，保持目录约定不变）
- `Storage/alluxio/doc/image-info.yml`: tags 表格首行版本 `2.9.6` 改回 `2.9.5`，Tag 改回 `2.9.5-oe2403sp4`
- `Storage/alluxio/README.md`: Supported tags 表格首行版本 `2.9.6` 改回 `2.9.5`，Tag 改回 `2.9.5-oe2403sp4`

## 修复逻辑
经核查上游（GitHub `Alluxio/alluxio`、`downloads.alluxio.io`、Docker Hub）确认：
- `v2.9.6` 仅为源码 tag（tag 日期 2024-07-02），**没有** GitHub Release、**没有** 二进制包，`downloads.alluxio.io/downloads/files/2.9.6/` 目录不存在（404）；
- 最新且有官方二进制包与官方镜像的版本为 **2.9.5**（`.../2.9.5/alluxio-2.9.5-bin.tar.gz` 返回 200；Docker Hub `alluxio/alluxio:2.9.5` 存在）。

同时发现本 PR 的基线（`origin/master`）中该目录 `Storage/alluxio/2.9.6/24.03-lts-sp4/` 实际构建的就是 **alluxio 2.9.5**（目录名 2.9.6 为既有历史命名，与内部版本号并不一致）。本 PR 的自动升级把 Dockerfile 与三处元数据中的版本号统一误改为 2.9.6，从而引入 404。

因此最小化修复 = 仅回退这 4 处版本标签到基线值 2.9.5。修复后 `git diff origin/master -- Storage/alluxio` 为空，工作区与已知可构建的基线状态完全一致，构建预计通过。未改动 `conf/`、`entrypoint.sh`、`libexec/` 等与本失败无关的文件，也未新增任何文件。

## 潜在风险
无。改动仅是将版本标签回退到基线已存在且可构建的 2.9.5；目录路径 `2.9.6/24.03-lts-sp4/` 与元数据 path 保持原状，未改变 CI 发现的路径结构。待上游正式发布 alluxio 2.9.6 二进制包后，可由后续自动升级重新升级。
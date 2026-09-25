# 修复摘要

## 修复的问题
将 parquet 1.18.1 容器镜像的源码下载 URL 由错误的 `apache-parquet-format-1.18.1` 制品路径修正为上游实际存在的 `apache-parquet-1.18.1` 制品路径，解决 Docker 构建时 curl 返回 404 导致的失败。

## 修改的文件
- `Bigdata/parquet/1.18.1/24.03-lts-sp4/Dockerfile`: 第 6 行下载地址前缀由 `apache-parquet-format-${VERSION}` 改为 `apache-parquet-${VERSION}`（目录名与文件名同步修改）。

## 修复逻辑
- 根因（分析报告「根因定位」）：新增 Dockerfile 用 `${VERSION}=1.18.1` 拼出 `https://archive.apache.org/dist/parquet/apache-parquet-format-1.18.1/apache-parquet-format-1.18.1.tar.gz`，该路径 404，制品不存在，`curl -f` 失败后 `tar` 连锁报错，RUN 退出码 2。
- 已实际访问上游归档站核实（对应分析报告「修复方向 1」）：
  - `https://archive.apache.org/dist/parquet/` 目录清单中不存在 `apache-parquet-format-1.18.1/`（该目录返回 404）。
  - 实际存在 `apache-parquet-1.18.1/`，其中包含 `apache-parquet-1.18.1.tar.gz`（约 1.4M）。
  - 直接用 curl 验证：原 URL 返回 `404`，修正后的 `https://archive.apache.org/dist/parquet/apache-parquet-1.18.1/apache-parquet-1.18.1.tar.gz` 返回 `200`。
- 1.18.1 属于 parquet 的 1.x（parquet-mr）版本线，制品命名前缀为 `apache-parquet-`，与 2.x 的 `apache-parquet-format-` 不同；本次仅针对 1.18.1 的 Dockerfile 修正前缀，未改动 2.11.0/2.12.0 等既有文件（其 URL 本身有效，且不在原始 PR 变更范围内）。
- 已确认压缩包顶层目录为 `apache-parquet-1.18.1/`，与现有 `tar --strip-components=1` 的解压方式兼容。
- 元数据文件（`meta.yml`、`README.md`、`doc/image-info.yml`）已由原 PR 正确注册 `1.18.1-oe2403sp4` 条目，与新修正后的 Dockerfile 一致，无需改动。

## 潜在风险
- 运行期风险（分析报告「需要进一步确认的点 4」）：`apache-parquet-1.18.1.tar.gz` 是 parquet-mr 源码包，源码内只有 `parquet-cli` 等模块，并不直接产出 `/usr/local/parquet/bin/parquet`，因此第 12 行 `ENTRYPOINT [ "parquet", "--version" ]` 在容器运行时可能报 command not found。此问题在现有 2.11.0/2.12.0 镜像中同样存在（既有仓库模式通病），且本次 CI 失败发生在构建阶段，故未扩大改动范围去引入 Maven/JDK 构建。若后续 CI 增加运行期校验，需要另行改造该镜像。
- `doc/image-info.yml` 中 `upstream.version_prefix: apache-parquet-format` 与 1.x 制品前缀不一致；但自动升级链路（`scripts/harness/query_version.py`）通过 anitya 获取版本，不依赖该字段，本次未改动以免影响既有 2.x 镜像的元数据语义。
- 未添加 Copyright/SPDX 头（分析报告点 3），因其为可能的后继检查点而非本次失败根因，遵循最小化原则未处理。
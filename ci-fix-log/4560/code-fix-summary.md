# 修复摘要

## 修复的问题
修正 parquet 1.18.1 镜像 Dockerfile 中错误的源码下载地址（上游制品前缀错用 `apache-parquet-format`，导致 HTTP 404，Docker 构建失败）。

## 修改的文件
- `Bigdata/parquet/1.18.1/24.03-lts-sp4/Dockerfile`: 第 6 行下载 URL 由 `apache-parquet-format-${VERSION}` 改为 `apache-parquet-${VERSION}`（目录名与 tar.gz 文件名同步修改），其余内容保持不变。

## 修复逻辑
- 分析报告指出 `VERSION=1.18.1` 展开后请求 `https://archive.apache.org/dist/parquet/apache-parquet-format-1.18.1/apache-parquet-format-1.18.1.tar.gz` 返回 404（`curl -f` 失败，后续 `tar` 因文件不存在再报错，exit code 2）。
- 经查实际上游归档目录 `https://archive.apache.org/dist/parquet/`：
  - `apache-parquet-1.18.1/` 存在；
  - `apache-parquet-format-1.18.1/` 不存在（该系列版本为 2.x，如 2.11.0/2.12.0/2.13.0/2.14.0）。
  - 即 `1.18.1` 属于 parquet-mr（`apache-parquet-*`）制品系列，而非 parquet-format 系列，因此下载前缀应为 `apache-parquet-`。
- 该修复对应分析报告“方向 1”：修正新增 Dockerfile 的源码下载地址，使其指向 `archive.apache.org` 上实际存在的归档，且保留 PR 预期的 `1.18.1` 版本（不改变版本号、目录名与元数据中的 tag）。
- 说明：现有 2.11.0/2.12.0 镜像使用 `apache-parquet-format-2.x` 能构建成功是因为对应 format 归档存在；本 PR 的 1.18.1 属于另一制品系列，故需使用 `apache-parquet-1.18.1`。

### URL 验证结果（提交前已实测）
- `curl -sIL https://archive.apache.org/dist/parquet/apache-parquet-1.18.1/apache-parquet-1.18.1.tar.gz` → `HTTP/1.1 200 OK`，`Content-Type: application/x-gzip`，`Content-Length: 1420964`。
- 实际下载后 `gzip -t` 校验通过，`tar -tzf` 顶层目录为 `apache-parquet-1.18.1/`，与 `--strip-components=1` 解压方式兼容。
- 对照：原 URL `apache-parquet-format-1.18.1` 请求 HTTP 状态码为 `404`。

## 潜在风险
- `Bigdata/parquet/doc/image-info.yml` 中 `upstream.version_prefix` 仍为 `apache-parquet-format`，与实际使用的 `apache-parquet` 制品系列不一致；这属于自动升级工具的版本来源配置，不影响本次 Docker 构建，但后续自动升级可能再次生成错误前缀的 Dockerfile。按“最小化修改”原则本次未改动该文件，建议后续单独修正上游配置（含 `version_url` 中 `aphache` 拼写）。
- 该镜像 `ENTRYPOINT [ "parquet", "--version" ]` 依赖解压产物中存在 `parquet` 可执行文件；parquet-mr 源码归档为 Java 源码工程，是否含可用 `parquet` 命令不在本次 CI 构建失败范围内，未做进一步验证。
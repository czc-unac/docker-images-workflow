# 修复摘要

## 修复的问题
修复 parquet 1.18.1 镜像 Dockerfile 中下载 URL 前缀错误导致的 HTTP 404（dependency-error），使构建可正常下载上游源码包。

## 修改的文件
- `Bigdata/parquet/1.18.1/24.03-lts-sp4/Dockerfile`: 第 6 行下载地址由 `apache-parquet-format-${VERSION}/apache-parquet-format-${VERSION}.tar.gz` 改回 `apache-parquet-${VERSION}/apache-parquet-${VERSION}.tar.gz`。

## 修复逻辑
- 根因：PR 将 URL 前缀改成了 `apache-parquet-format-`，但 `1.18.1` 属于 parquet-java（`apache-parquet`）版本线，`apache-parquet-format` 格式库只有 2.x（2.3.1～2.14.0）版本，不存在 `1.18.1`，因此 `https://archive.apache.org/dist/parquet/apache-parquet-format-1.18.1/...` 返回 404。
- 已从上游 archive.apache.org 验证：`apache-parquet-format-1.18.1` 返回 404；`apache-parquet-1.18.1/apache-parquet-1.18.1.tar.gz` 返回 200（可正常下载）。
- 该修复等价于把被本次 PR 改坏的那一行还原为仓库历史中已验证可用的写法（git blob `5b4493c18`），与此前针对同一文件的 CI 修复（commit `5db91a953`，PR #4560）完全一致。
- 其余三个文件（README.md、doc/image-info.yml、meta.yml）相对于基线未发生实质变更，其 1.18.1 版本/tag 元数据仍自洽，故不做修改，保持最小化。

## 潜在风险
无。仅恢复一行下载地址，未改变版本号、目录结构或镜像标签；不影响 2.11.0、2.12.0 等使用 `apache-parquet-format` 前缀的既有镜像。
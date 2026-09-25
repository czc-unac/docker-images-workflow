# 修复摘要

## 修复的问题
修复 diskann 5.0.3 镜像构建时 `git clone` 使用了不存在的 ref `v5.0.3` 导致 exit code 128 的构建失败。

## 修改的文件
- `AI/diskann/5.0.3/24.03-lts-sp4/Dockerfile`: 将 `git clone -b v${VERSION}` 改为 `git clone -b diskann-garnet-v${VERSION}`，使 ref 与实际上游 tag `diskann-garnet-v5.0.3` 一致。

## 修复逻辑
分析报告根因指出 `ARG VERSION=5.0.3` 配合 `git clone -b v${VERSION}` 构造出的 `v5.0.3` 在 `microsoft/DiskANN` 中不存在。已实际查询上游 refs 验证：
- `git ls-remote --tags https://github.com/microsoft/DiskANN.git` 确认 5.0.3 对应的真实 tag 为 `diskann-garnet-v5.0.3`（同系列还有 `diskann-garnet-v5.0.0`~`v5.0.3`），不存在 `v5.0.3`。
- 该 tag 确实存在：`da5b7323cd9cab702b9cde8e4620b149cce6928f refs/tags/diskann-garnet-v5.0.3`。
- 实际以 `diskann-garnet-v5.0.3` 浅克隆上游仓库成功，并确认工作区内二进制目标齐全：`diskann-benchmark`、`compute_groundtruth`、`compute_multivec_groundtruth`、`gen_associated_data_from_range`、`generate_minmax`、`generate_pq`、`generate_synthetic_labels`、`random_data_generator`、`relative_contrast`、`subsample_bin`，与 Dockerfile 中 COPY 列表完全匹配。

因此仅需修正 ref 前缀：保留 `VERSION=5.0.3`（与镜像 tag `5.0.3-oe2403sp4`、`meta.yml`、`README.md`、`doc/image-info.yml` 中的 5.0.3 一致），在 clone 的 ref 模板上补上 `diskann-garnet-` 前缀，即可得到真实存在的 `diskann-garnet-v5.0.3`。

## 潜在风险
无。改动仅影响该新版本 Dockerfile 的 clone ref，未改动 `VERSION` 值及文档中引用的 5.0.3 版本号，不影响其他镜像；后续自动升级沿用本 Dockerfile 作为模板时也会持续使用正确前缀。
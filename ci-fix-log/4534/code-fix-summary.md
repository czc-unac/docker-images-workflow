# 修复摘要

## 修复的问题
Dockerfile 中 `git clone -b v${VERSION}` 构造出的 ref 为 `v5.0.3`，该 tag 在上游 `microsoft/DiskANN` 不存在，导致 clone 失败（exit code 128）。已将其修正为上游实际存在的 `diskann-garnet-v${VERSION}`。

## 修改的文件
- `AI/diskann/5.0.3/24.03-lts-sp4/Dockerfile`: 第 13 行 `git clone -b v${VERSION} ...` 改为 `git clone -b diskann-garnet-v${VERSION} ...`

## 修复逻辑
- 分析报告根因（build-error）：`AI/diskann/5.0.3/24.03-lts-sp4/Dockerfile:13` 使用 `v${VERSION}`，在 `VERSION=5.0.3` 时得到 `v5.0.3`，上游仓库无此 ref，git 返回 `fatal: Remote branch v5.0.3 not found in upstream origin`。
- 已通过 `git ls-remote --tags https://github.com/microsoft/DiskANN.git` 确认上游 tag 命名：DiskANN 5.0.3 对应的 release tag 为 `diskann-garnet-v5.0.3`（GitHub Release 名称为 `diskann-garnet v5.0.3`），其 ref 值为 `refs/tags/diskann-garnet-v5.0.3`（commit `da5b7323cd9cab702b9cde8e4620b149cce6928f`），确实不存在 `v5.0.3`。
- 已从上游 `diskann-garnet-v5.0.3` tag 拉取源码验证：该 tag 根目录包含 Cargo workspace，`diskann-garnet/Cargo.toml` 的 version 为 `5.0.3`，与镜像/文档版本号语义一致；且 `diskann-benchmark`（diskann-benchmark 包）与 `diskann-tools/src/bin/` 下的 `compute_groundtruth`、`compute_multivec_groundtruth`、`gen_associated_data_from_range`、`generate_minmax`、`generate_pq`、`generate_synthetic_labels`、`random_data_generator`、`relative_contrast`、`subsample_bin` 均存在，Dockerfile 中 `cargo build --release --workspace` 与全部 `COPY .../target/release/<tool>` 目标可满足。
- 验证新 ref 可命中：`git ls-remote --tags ... 'refs/tags/diskann-garnet-v5.0.3'` 返回成功；并以 `git clone -b diskann-garnet-v5.0.3 --depth 1` 实际克隆成功。
- 采用最小改动：仅调整 clone 的 `-b` 参数前缀，保留 `ARG VERSION=5.0.3` 及 README/image-info.yml/meta.yml 中的 `5.0.3-oe2403sp4` 版本标识不变，保持镜像标签与文档一致，且对未来 `diskann-garnet-v5.0.x` 发布同样适用。

## 潜在风险
- 无。改动仅影响该新增镜像的 git ref 构造；不涉及其他既有镜像与文件。
- 说明：上游 `rust-toolchain.toml` 固定 `channel = "1.97.1"`，构建时 rustup 会按该文件下载 1.97.1（Dockerfile 中的 `RUST_VERSION=1.92.0` 仅为默认工具链）。该行为在既有 0.59.0 镜像中一致，且当前 CI 失败发生在 clone 步骤（第 9 步）而非工具链步骤，非本次根因，未做改动。
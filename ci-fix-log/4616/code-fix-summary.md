# 修复摘要

## 修复的问题
修复 diskann 5.0.3 镜像构建时 `git clone` 引用不存在的 tag `v5.0.3` 导致 `exit code 128` 的问题。

## 修改的文件
- `AI/diskann/5.0.3/24.03-lts-sp4/Dockerfile`: 将克隆 ref 由 `v${VERSION}` 改为上游实际存在的 tag 命名 `diskann-garnet-v${VERSION}`（即 `diskann-garnet-v5.0.3`）。

## 修复逻辑
CI 失败根因是 Dockerfile 第 13 行 `git clone -b v${VERSION}` 展开为 `v5.0.3`，而 `microsoft/DiskANN` 上游不存在该 tag/分支。

已通过上游仓库实际校验：
- 执行 `git ls-remote --tags https://github.com/microsoft/DiskANN.git`，完整 tag 列表中不存在 `v5.0.3`，唯一的 5.0.3 tag 为 `diskann-garnet-v5.0.3`。
- 通过 GitHub Releases API 确认 `diskann-garnet-v5.0.3`（release 名 `diskann-garnet v5.0.3`，发布于 2026-09-22）是上游最新 release，自动升级工具据其解析出 `VERSION=5.0.3`。
- 单独校验 `git ls-remote https://github.com/microsoft/DiskANN.git refs/tags/diskann-garnet-v5.0.3` 成功解析（`da5b7323...`，exit=0），确认 `git clone -b diskann-garnet-v5.0.3 --depth 1` 可正常克隆。
- 校验该 tag 的 workspace 结构：`diskann-tools/src/bin/` 下存在 `compute_groundtruth.rs`、`compute_multivec_groundtruth.rs`、`gen_associated_data_from_range.rs`、`generate_minmax.rs`、`generate_pq.rs`、`generate_synthetic_labels.rs`、`random_data_generator.rs`、`relative_contrast.rs`、`subsample_bin.rs`，且 `diskann-benchmark` crate 存在并产出同名二进制，与 Dockerfile 后续 `COPY --from=builder /build/target/release/...` 引用的 10 个二进制一致，`cargo build --release --workspace` 可产出全部目标文件。

修改仅涉及克隆 ref 一个 token，未改动 `ARG VERSION=5.0.3` 及 README、meta.yml、image-info.yml 中的版本语义，保持最小化。

## 潜在风险
- `diskann-garnet-v5.0.3` 是上游针对 garnet 组件的 release tag，其 workspace 版本为 `0.59.0`（与已有 0.59.0 镜像代码高度重合）。该 5.0.3 镜像在语义上对应的是 DiskANN 0.59.0 代码库 + garnet 5.0.3 修复，而非一个独立的 DiskANN 5.0.3 大版本。
- 自动升级工具当前会从 `diskann-garnet-vX.Y.Z` 这类组件 tag 中解析出版本号，后续若上游发布新的主版本（如 `v0.60.0`），本 Dockerfile 的 `diskann-garnet-v${VERSION}` 前缀可能不再适用，需届时同步调整；本次仅针对 5.0.3 修复。
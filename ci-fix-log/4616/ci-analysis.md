# CI 失败分析报告

## 基本信息
- PR: #4616 — 【自动升级】diskann容器镜像升级至5.0.3版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 模式22
- 新模式标题: (不适用，已匹配模式22)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#9 [builder 4/6] RUN git clone -b v5.0.3 --depth 1 https://github.com/microsoft/DiskANN.git /build
#9 0.188 Cloning into '/build'...
#9 0.696 fatal: Remote branch v5.0.3 not found in upstream origin
#9 ERROR: process "/bin/sh -c git clone -b v${VERSION} --depth 1 https://github.com/microsoft/DiskANN.git /build" did not complete successfully: exit code: 128
------
Dockerfile:13
--------------------
  13 | >>> RUN git clone -b v${VERSION} --depth 1 https://github.com/microsoft/DiskANN.git /build
```

### 根因定位
- 失败位置: `AI/diskann/5.0.3/24.03-lts-sp4/Dockerfile:13`
- 失败原因: Dockerfile 中 `VERSION=5.0.3`，`git clone -b v${VERSION}` 展开为分支/tag `v5.0.3`，而 `microsoft/DiskANN` 上游仓库中不存在名为 `v5.0.3` 的分支或 tag（`fatal: Remote branch v5.0.3 not found in upstream origin`），git 克隆以 exit code 128 失败，导致 Docker 构建终止。

### 与 PR 变更的关联
本次 PR 新增了 `AI/diskann/5.0.3/24.03-lts-sp4/Dockerfile`（见 `pr.diff`，新文件，31 行新增）。该文件第 13 行的 clone 引用是全新引入的，上游不存在 `v5.0.3`，因此失败由本次 PR 直接触发。README.md、`doc/image-info.yml`、`meta.yml` 的改动（新增 5.0.3 条目）为配套元数据更新，不是失败原因。

## 修复方向

### 方向 1（置信度: 高）
`git clone -b` 引用的 tag/分支名与 `microsoft/DiskANN` 上游实际存在的名称不一致（当前带 `v` 前缀的 `v5.0.3` 不存在）。需要核对上游仓库 `microsoft/DiskANN` 中 5.0.3 对应 release 的实际 tag 命名（可能是无 `v` 前缀的 `5.0.3`，或该版本号根本不存在/命名不同），将 clone 的 ref 改为与上游一致的 tag/分支名。

### 方向 2（置信度: 中）
若上游 5.0.3 实际以 release 制品而非 git tag 形式提供，则需要改为从对应 release 下载源码包，或改用存在的 tag。此方向需先确认上游制品发布方式。

## 需要进一步确认的点
1. `microsoft/DiskANN` 上游仓库中 5.0.3 版本对应的**准确 tag 名称**：是否存在 `5.0.3`（无 `v` 前缀）、`v5.0.3`、或其他命名；抑或 5.0.3 版本号本身有误（历史版本为 0.52.0、0.59.0，5.0.3 为跨大版本号，需确认版本号来源）。
2. 该 clone 后是否还有其他针对 5.0.3 的假设（如 `cargo build --workspace` 产出的二进制名 `diskann-benchmark` 等）与目标 tag 的实际 workspace 结构是否一致。
3. 需确认此失败是否同时存在于 aarch64 job（当前日志为 x86_64 builder 阶段的失败，属真实构建错误，非编排层误报；日志末尾为 `Finished: FAILURE`，与 `ci_failed` 状态一致）。

## 修复验证要求
本修复方向涉及修改 Dockerfile 中对上游 git ref 的引用（非正则 patch 外部源文件），code-fixer 在提交前应：
1. 从上游 `microsoft/DiskANN` 仓库确认 5.0.3 对应的实际 tag/分支名（例如通过 GitHub API 或 tags 列表核对），验证修正后的 `-b` 参数能在该仓库成功解析。
2. 确认修正后 tag 下的 workspace 确实能 build 出 Dockerfile 后续 `COPY --from=builder /build/target/release/...` 所引用的 10 个二进制文件。

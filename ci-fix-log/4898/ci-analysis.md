# CI 失败分析报告

## 基本信息
- PR: #4898 — 【自动升级】seurat容器镜像升级至5.6.0版本.
- 失败类型: `infra-error`（实际为：证据不足，日志缺失）
- 置信度: 低
- 知识库匹配: 模式42（日志缺失无法定位）
- 新模式标题: 镜像内容错配
- 新模式症状关键词: seurat, pblat-cluster, Dockerfile内容与镜像名不符, 自动升级

## 根因分析

### 直接错误
```
ci.run_info: "(not available)"
ci.logs: "(not available — analyze based on PR diff only)"
```

本次**未提供任何 CI 日志与运行信息**，无法从日志中提取第一条真实错误。

### 前置检查（日志与状态一致性）
- 日志中**不存在** `Finished: SUCCESS` / `Build successful` 成功标志；
- 但同时也**根本没有任何日志内容**，因此无法判断失败发生在 trigger/编排层还是下游架构构建 job（`/job/x86-64/…`、`/job/aarch64/…`），也无法判断真实错误类型。

结论：**证据不足**，无法定位具体失败点。

### 根因定位
- 失败位置: 未知（日志缺失）
- 失败原因: 无法确认。仅凭 diff 无法判定 CI 究竟因何失败。

### 与 PR 变更的关联（diff 层面的重大异常）
对 `pr.diff` 逐项核对时发现一处强烈异常，虽不能直接证明它是本次 CI 失败原因，但极可能是本 PR 的核心缺陷：

新增文件 `HPC/seurat/5.6.0/24.03-lts-sp4/Dockerfile` 的**内容与镜像名完全不符**：

- PR 标题、`HPC/seurat/README.md`、`HPC/seurat/doc/image-info.yml`、`HPC/seurat/meta.yml` 均将该条目声明为 **Seurat 5.6.0**；
- 但 Dockerfile 实际内容是构建 **pblat-cluster**：
  - `ARG VERSION=1.1`（与 seurat 5.6.0 不匹配）；
  - `git clone -b ${VERSION} https://github.com/icebert/pblat-cluster.git /pblat-cluster`；
  - 安装 `git gcc gcc-c++ make which zlib-devel openssl-devel openmpi-devel`，最后 `cp pblat-cluster /usr/bin/pblat`。
- 这与 Seurat（R 语言单细胞分析包，通常经 R/CRAN 安装）毫无关系，明显是把**另一个镜像的构建内容误写入了 seurat 路径**。

此外该 Dockerfile 还存在可疑点（在无日志时不能确认是否为直接原因）：
- 硬编码 aarch64 的 sed：`sed -i 's/x86_64/aarch64/g' htslib/Makefile`，在 amd64 构建上会构造错误架构；
- `ENV PATH=/usr/lib64/openmpi/bin:$PATH` 自引用未定义变量（模式20 特征）。

## 修复方向

### 方向 1（置信度: 中）
核对 `HPC/seurat/5.6.0/24.03-lts-sp4/Dockerfile` 的预期内容：当前文件内容是 pblat-cluster 构建脚本，与“seurat 5.6.0 升级”不符，判断为自动升级流程写入了错误的 Dockerfile 内容。应使用与其它 seurat 版本（如 5.5.0/5.5.1）一致的 Seurat 安装逻辑重写该文件，并同步确认 `ARG VERSION` 指向正确的 seurat 版本。

### 方向 2（置信度: 低）
若该 Dockerfile 内容确为预期，则需获取真实日志后定位 pblat-cluster 构建失败点，例如上游 `icebert/pblat-cluster` 是否存在 tag `1.1`（diff 中 `ARG VERSION=1.1`）、`make` 是否因架构 sed 或 `-fcommon` 补丁失败等。

## 需要进一步确认的点
1. **获取真实 CI 日志**：trigger 层日志，以及下游架构构建 job（`/job/x86-64/…`、`/job/aarch64/…`）的日志，才能确定真正的失败类型与第一条错误。
2. 确认 `HPC/seurat/5.6.0/24.03-lts-sp4/Dockerfile` 期望内容是否应为 Seurat 安装（对照 5.5.0 / 5.5.1 版本），当前 pblat-cluster 内容是否为误写入。
3. 确认上游 `icebert/pblat-cluster` 仓库是否存在 tag/branch `1.1`。
4. 确认 CI 是否按元数据（`meta.yml`）调度到 amd64 与 arm64 两个架构，以及是否因硬编码 aarch64 的 sed 在 amd64 上产生失败。

## 修复验证要求
（本次修复方向不涉及“修改正则匹配第三方源文件”，无需执行正则验证。但 Code Fixer 在改动前必须先获得真实 CI 日志，或至少核对 seurat 5.5.x 既有 Dockerfile 的实际构建方式，不能仅凭本报告假设内容错配一定成立。）

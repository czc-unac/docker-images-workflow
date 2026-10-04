# CI 失败分析报告

## 基本信息
- PR: #4907 — 【自动升级】lammps容器镜像升级至2026.09.30版本.
- 失败类型: build-error（暂定，证据不足）
- 置信度: 低
- 知识库匹配: 模式42（与模式02 高度相关）
- 新模式标题: (非新模式)
- 新模式症状关键词: (非新模式)

> 说明：本次上下文 `ci.logs` 为 `(not available — analyze based on PR diff only)`，
> 没有任何构建日志可供比对。因此下面所有结论均来自 PR diff 与历史知识库，属于**推断**，
> 不能视为已定性的根因。请务必先补齐失败 job 日志再交由 Code Fixer 处理。

## 根因分析

### 直接错误
无日志可引用。`ci.logs` 未提供，无法复制任何实际报错行。
唯一可从 diff 中定位的可疑下载逻辑为新增 Dockerfile 中的：

```
RUN wget https://github.com/lammps/lammps/archive/refs/tags/stable_${VERSION}.tar.gz \
    && tar -zxvf stable_${VERSION}.tar.gz \
    && rm -f stable_${VERSION}.tar.gz
```

其中 `ARG VERSION=2026.09.30`，展开后请求
`https://github.com/lammps/lammps/archive/refs/tags/stable_2026.09.30.tar.gz`。

### 根因定位
- 失败位置: `HPC/lammps/2026.09.30/24.03-lts-sp4/Dockerfile`（wget 下载步骤，证据不足）
- 失败原因: 证据不足。基于知识库推断，最可能是上游 LAMMPS 不存在
  `stable_2026.09.30` 这一 tag，导致 wget 下载 404 / tar 解压失败。
  但当前无任何日志可验证。

### 与 PR 变更的关联
本 PR 为自动升级 PR，新增 `2026.09.30` 版本目录及其 Dockerfile，并在
`meta.yml`、`README.md`、`doc/image-info.yml` 中登记新 tag。
若失败发生，最可能由新增 Dockerfile 中的 `ARG VERSION=2026.09.30` 及其构造的
`stable_${VERSION}` 下载 URL 触发。

**重要历史佐证**：知识库模式42 已收录同一路径同一版本的先例
`PR #4861: HPC/lammps/2026.09.30/24.03-lts-sp4/Dockerfile`，其结论为
“LAMMPS 自动升级 PR 使用了不存在的上游 tag `stable_2026.09.30`，导致 Dockerfile 下载失败”。
本 PR #4907 与之形态完全一致，高度疑似同一问题的重复提交。

## 修复方向

### 方向 1（置信度: 低）
核对上游 `lammps/lammps` 仓库在对应时间点实际存在的 release tag
（LAMMPS 历史 tag 形如 `stable_29Aug2024`、`stable_22Jul2025`，并非 `stable_<YYYY.MM.DD>` 形式），
将 `ARG VERSION` 及目录名/README/meta.yml/image-info.yml 中的版本统一修正为上游真实存在的 tag 值。
在日志补齐前，此方向仅为推断。

### 方向 2（可选，置信度: 低）
若上游确实提供 `2026.09.30` 但 tar 包地址为 GitHub archive 而非常规 release 制品，
则需确认 GitHub `archive/refs/tags/` 路径与 tag 命名是否匹配；此方向同样需要日志佐证。

## 需要进一步确认的点
1. **必须获取失败 job 的实际构建日志**（本上下文未提供 `ci.logs`）。需要确认失败发生在
   x86-64 还是 aarch64 架构 job，以及是否出现在 `wget` 下载、`tar` 解压、`make mpi` 或
   `dnf install` 等具体步骤。
2. 确认上游 `lammps/lammps` 是否存在 `stable_2026.09.30` tag。
3. 确认 CI 是否已有 `ci_failed` 标签及对应 run 记录；`ci.run_info` 本次为
   `(not available)`。
4. 核对该自动升级 PR 生成逻辑：`2026.09.30` 是否被上游 tag 命名规则接受，还是自动升级工具
   误用了日期格式。

## 修复验证要求
- 本 PR 尚无日志支撑，置信度为**低**，**不应在未获取失败日志前直接套用上述修复方向**。
- Code Fixer 在提交前必须完成以下验证：
  1. 从 CI 失败 job（如 `/job/x86-64/…` 或 `/job/aarch64/…`）获取真实错误日志，
     确认首个 error 所在步骤。
  2. 以上游仓库为准，核对 `Dockerfile` 中 `ARG VERSION` 展开后的
     `stable_${VERSION}` 是否对应真实存在的 tag；若修复涉及修改版本号，
     必须以 `lammps/lammps` 仓库（或其 release/tag 列表）实际存在的内容为准，
     确认下载 URL 可访问后再提交。
  3. 同步更新 `meta.yml`、`README.md`、`doc/image-info.yml` 中的版本条目，保持四处一致。

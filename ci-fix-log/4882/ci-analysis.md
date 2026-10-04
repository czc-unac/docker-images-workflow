# CI 失败分析报告

## 基本信息
- PR: #4882 — 【自动升级】openfoam容器镜像升级至20260907版本.
- 失败类型: dependency-error
- 置信度: 中
- 知识库匹配: 模式02（下载 URL 硬编码版本路径错误 / 软件包版本不存在）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
本次上下文 **未提供任何 CI 日志**（`ci.logs` = "not available — analyze based on PR diff only"），
因此无法引用日志中的原始错误行。基于 PR diff 与知识库中针对本 PR 的记录，直接错误推断为
OpenFOAM 源码包下载 404：

```
RUN wget https://sourceforge.net/projects/openfoam/files/v${VERSION}/ThirdParty-v${VERSION}.tgz && \
    wget https://sourceforge.net/projects/openfoam/files/v${VERSION}/OpenFOAM-v${VERSION}.tgz
```

由于 `ARG VERSION=20260907`，上述 URL 展开为
`.../files/v20260907/OpenFOAM-v20260907.tgz`，该路径在上游不存在。

### 根因定位
- 失败位置: `HPC/openfoam/20260907/24.03-lts-sp4/Dockerfile:9`（wget 下载步骤）
- 失败原因: 自动升级写入的版本号 `20260907` 并非 OpenFOAM 上游真实发布的版本。
  OpenFOAM 的版本号为 4 位 `YYMM` 形式（与 `doc/image-info.yml` 中 `regex: (?i)v\d{4}\b`、
  以及既有标签 `2506`、`2412` 一致），`20260907`（8 位）不符合该格式，上游不存在
  `v20260907` 目录，导致 `ThirdParty-v20260907.tgz` / `OpenFOAM-v20260907.tgz` 下载返回 404，Docker 构建失败。

### 与 PR 变更的关联
强关联。本 PR 为自动升级单，新增：
1. `HPC/openfoam/20260907/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=20260907` 直接决定了下载 URL；
2. `HPC/openfoam/README.md`、`HPC/openfoam/doc/image-info.yml` 新增 `20260907-oe2403sp4` 标签行；
3. `HPC/openfoam/meta.yml` 新增 `20260907-oe2403sp4` 条目。

版本号错误即由该 PR 引入，失败与本次改动直接相关。

## 修复方向

### 方向 1（置信度: 中）
将 `VERSION` 修正为 OpenFOAM 上游真实存在的版本号（4 位 `YYMM` 形式），并同步修改所有引用该
版本号的位置：`Dockerfile` 中的 `ARG VERSION`、`meta.yml` 的标签与路径、`README.md` 与
`doc/image-info.yml` 中的标签/链接（含 URL 中的 `20260907` 目录）。知识库记录显示本 PR 对应的
正确版本为 `2606`，但应以实际上游发布为准。

### 方向 2（可选，置信度: 低）
若上游确实未发布目标版本，则需要人工确认应升级到哪个可用版本，而非由自动升级脚本写入日期串。

## 需要进一步确认的点
1. **CI 日志缺失**：本次未提供失败 job 日志，无法从日志确认失败的确切命令与退出码（推断为
   `wget` 404 / `ERROR 404: Not Found`）。需获取真正失败的下游构建 job 日志
   （如 `/job/x86-64/…`、`/job/aarch64/…`）进行核对。
2. **上游真实版本号**：需确认 OpenFOAM 在 sourceforge
   `project/openfoam/files/` 下当前实际存在的 `vYYMM` 版本目录（知识库指向 `2606`）。
3. **自动升级来源**：需确认自动升级脚本为何将版本解析为 `20260907`（疑似把日期当作版本），
   以避免后续再次生成不存在的版本号。

## 修复验证要求
本修复不涉及对第三方源文件正则 patch，无需从上游拉取文件验证正则。
但 code-fixer 在提交前必须确认目标版本号在 OpenFOAM 上游下载目录中真实存在，
避免修复后仍为 404。

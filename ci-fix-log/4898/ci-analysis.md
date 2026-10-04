# CI 失败分析报告

## 基本信息
- PR: #4898 — 【自动升级】seurat容器镜像升级至5.6.0版本.
- 失败类型: build-error（证据不足，无法唯一确定）
- 置信度: 低
- 知识库匹配: 模式42（日志缺失无法定位）
- 新模式标题: （不适用，匹配已有模式）
- 新模式症状关键词: （不适用）

## 根因分析

### 直接错误
本次上下文未提供任何有效 CI 日志：

```
ci.run_info: (not available)
ci.logs: (not available — analyze based on PR diff only)
```

因此**不存在可用于定位根因的错误行**。核心约束规定：日志不足以确定根因时必须判定为"证据不足"。以下所有内容均为基于 `pr.diff` 的**可疑点假设**，未经日志验证，不构成确定结论。

### 根因定位
- 失败位置: 未知（无日志）
- 失败原因: 无法确认。日志缺失，无法判断失败发生在构建、预检（license/metadata 校验）还是下游架构任务中。

### 与 PR 变更的关联
PR 新增/修改了 4 处内容：
1. 新增 `HPC/seurat/5.6.0/24.03-lts-sp4/Dockerfile`（新文件，19 行）
2. `HPC/seurat/README.md` 增加 5.6.0 行
3. `HPC/seurat/doc/image-info.yml` 增加 5.6.0 行
4. `HPC/seurat/meta.yml` 增加 `5.6.0-oe2403sp4` 条目

**可疑点 A（架构硬编码，build-error 假设，中低置信度）**：
新增 Dockerfile 中 sed 替换无条件把 `x86_64` 改成 `aarch64`：
```
sed -i 's/MACHTYPE=x86_64/MACHTYPE=aarch64/g' Makefile
sed -i 's/x86_64/aarch64/g' htslib/Makefile
```
而 README/image-info 声明该镜像支持 `amd64, arm64`。若 CI 在 amd64 runner 上构建同一 Dockerfile，上述替换会把 x86_64 平台的构建配置强制改成 aarch64，极可能导致 x86_64 构建失败。**但无日志佐证，仅为假设。**

**可疑点 B（语义内容与标题不符）**：
PR 标题为"seurat 升级至 5.6.0"，但新增 Dockerfile 实际构建的是 `icebert/pblat-cluster`（`ARG VERSION=1.1`，`cp pblat-cluster /usr/bin/pblat`）。即 seurat 的路径下放入的是 pblat-cluster 的构建脚本。这是自动升级单的明显内容错配，但 CI 是否对该语义做校验未知，无法据此断定失败。

**可疑点 C（license 头缺失，lint-error 假设，中低置信度）**：
新增 Dockerfile 直接从 `ARG BASE=...` 开始，未包含项目要求的 Copyright + SPDX 头（参考模式17 `check_package_license`）。若 CI 对此新文件执行 license 检查，会判定失败。同样无日志佐证。

## 修复方向

> 因日志缺失，以下均为待验证的排查方向，**不构成确定结论**。

### 方向 1（置信度: 低）— 获取失败 job 日志
确认真正失败的 job 与步骤（预检/架构构建/推送）。在拿到日志前，不应据 PR diff 假定根因。

### 方向 2（置信度: 中低）— 校验新增 Dockerfile 的架构处理
检查 sed 对 `x86_64/aarch64` 的硬编码替换是否导致 amd64 构建失败；若确实如此，应按目标架构做条件分支（仅推测，无日志确认）。

### 方向 3（置信度: 中低）— 校验新文件 license 头
检查 `check_package_license` 是否为该新增 Dockerfile 的失败来源。

## 需要进一步确认的点
1. **失败 job 的真实日志**：本次仅提供 PR diff，无 `ci.logs`。需提供失败 job（预检、x86-64、aarch64、推送等）的完整日志才能定位根因。
2. **失败类型归属**：无法区分是 build-error、lint-error（license/metadata 校验）还是 infra-error。
3. **CI 是否校验 Dockerfile 内容与镜像名的一致性**：即 pblat-cluster 内容放在 seurat 路径下是否会被 CI 拦截。
4. **meta.yml / image-info.yml 的一致性校验是否触发**：新增 5.6.0 条目是否满足项目的路径与 schema 规范（本次无法读取仓库文件验证）。
5. **amd64 架构构建是否确实因 sed 硬编码 aarch64 而失败**：需 x86-64 job 日志确认。

## 修复验证要求
本次分析置信度为"低"，且失败日志完全缺失。**code-fixer 在收到本报告后不得直接依据 diff 假设提交修复**，必须：
1. 先取得失败 job 的真实 `ci.logs`（尤其是下游架构构建 job），确认第一条 error。
2. 若确认失败源于 sed 架构硬编码，需分别验证 amd64 与 aarch64 目标下替换结果是否符合上游 Makefile 预期。
3. 若确认失败源于 license 头缺失，需比对仓库内同类（如 `HPC/seurat/5.5.1/.../Dockerfile`）既有文件的头部格式后再补充。
4. 本报告不涉及正则匹配第三方源文件的修复，故无 getdeps fetcher.py 类验证要求。

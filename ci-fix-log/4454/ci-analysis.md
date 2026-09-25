# CI 失败分析报告

## 基本信息
- PR: #4454 — 【自动升级】logstash容器镜像升级至9.5.4版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式06
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#15 [11/13] COPY env2yaml/env2yaml-amd64 /usr/local/bin/env2yaml
#15 ERROR: failed to calculate checksum of ref ye2e7g88qoaq8t76s3ewkkep3::ud5r0mybe56uhnalrrsorxzkw: "/env2yaml/env2yaml-amd64": not found

Dockerfile:39
--------------------
  37 |     COPY config/pipelines.yml config/log4j2.properties config/log4j2.file.properties config/
  38 |     COPY config/default.conf pipeline/logstash.conf
  39 | >>> COPY env2yaml/env2yaml-${TARGETARCH} /usr/local/bin/env2yaml
  40 |
  41 |     RUN chown --recursive logstash:root config/ pipeline/
--------------------
ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref ye2e7g88qoaq8t76s3ewkkep3::ud5r0mybe56uhnalrrsorxzkw: "/env2yaml/env2yaml-amd64": not found
```

### 根因定位
- 失败位置: `Bigdata/logstash/9.5.4/24.03-lts-sp4/Dockerfile:39`
- 失败原因: Dockerfile 中 `COPY env2yaml/env2yaml-${TARGETARCH} /usr/local/bin/env2yaml` 引用的二进制文件 `env2yaml/env2yaml-amd64`（以及 `env2yaml/env2yaml-arm64`）未随本次 PR 一并提交到仓库，BuildKit 计算 checksum 时找不到源文件。

### 与 PR 变更的关联
本 PR 新增了完整的 logstash 9.5.4 镜像目录，`pr.diff` 中只包含 `Dockerfile`、`config/`、`entrypoint.sh`、`README.md`、`image-info.yml`、`meta.yml`，**未包含 `env2yaml/` 目录下的任何二进制文件**。而 Dockerfile 第 39 行明确依赖 `env2yaml/env2yaml-${TARGETARCH}`，因此该 COPY 指令必然失败。这是本次 PR 直接引起的失败，与基础设施无关。

## 修复方向

### 方向 1（置信度: 高）
在 `Bigdata/logstash/9.5.4/24.03-lts-sp4/env2yaml/` 目录下补充提交两个架构的 `env2yaml` 二进制文件：`env2yaml-amd64` 与 `env2yaml-arm64`。可参考同仓库历史 logstash 版本（如 `9.4.0/24.03-lts-sp4/`、`9.3.4/`）中已有的 `env2yaml` 二进制获取方式（通常从官方 logstash 镜像中提取），确保与 Dockerfile 中 `COPY env2yaml/env2yaml-${TARGETARCH}` 路径、文件名完全一致，且具备可执行权限。

### 方向 2（备选，置信度: 中）
若上游 logstash 9.5.4 官方镜像/发行包已改变 env2yaml 的提供方式，则需同步调整 Dockerfile 第 39 行的 COPY 来源以匹配实际存在的文件；但当前证据指向"仓库遗漏二进制"而非路径设计错误，优先采用方向 1。

## 需要进一步确认的点
- 确认仓库中历史 logstash 版本目录下 `env2yaml/` 二进制的来源与获取方式，保证 9.5.4 所用二进制与版本匹配。
- 确认 `env2yaml-amd64` 与 `env2yaml-arm64` 两个文件均需提交（Dockerfile 通过 `TARGETARCH` 在两种架构上分别引用）。
- 日志末尾的 `LegacyKeyValueFormat: "ENV key=value" should be used instead of legacy "ENV key value" format (line 31)` 仅为 BuildKit 非致命警告，不是本次失败根因，无需处理。

## 修复验证要求
本修复不涉及正则 patch 外部源文件，无需上游源码验证。code-fixer 提交前须确认 `env2yaml/env2yaml-amd64`、`env2yaml/env2yaml-arm64` 两个文件确实被纳入本 PR 提交（而非本地未跟踪状态），否则 COPY 仍会失败。

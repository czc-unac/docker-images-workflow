# CI 失败分析报告

## 基本信息
- PR: #4619 — 【自动升级】logstash容器镜像升级至9.5.4版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式06
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#16 [11/13] COPY env2yaml/env2yaml-amd64 /usr/local/bin/env2yaml
#16 ERROR: failed to calculate checksum of ref gt0k50hxvgeomnmcc2b3wrl1g::xxhk0jwj6w0ua3d7b62u5m40k: "/env2yaml/env2yaml-amd64": not found

Dockerfile:39
--------------------
  37 |     COPY config/pipelines.yml config/log4j2.properties config/log4j2.file.properties config/
  38 |     COPY config/default.conf pipeline/logstash.conf
  39 | >>> COPY env2yaml/env2yaml-${TARGETARCH} /usr/local/bin/env2yaml
--------------------
ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref gt0k50hxvgeomnmcc2b3wrl1g::xxhk0jwj6w0ua3d7b62u5m40k: "/env2yaml/env2yaml-amd64": not found
```
（日志末尾为 `Finished: FAILURE`，非成功标志，前置检查通过，进入正式分析。）

### 根因定位
- 失败位置: `Bigdata/logstash/9.5.4/24.03-lts-sp4/Dockerfile:39`
- 失败原因: Dockerfile 中 `COPY env2yaml/env2yaml-${TARGETARCH} /usr/local/bin/env2yaml` 引用的 env2yaml 二进制文件未随本 PR 提交到仓库，BuildKit 在计算 COPY 源文件校验和时找不到 `env2yaml/env2yaml-amd64`。

### 与 PR 变更的关联
- 本 PR 新增了 `Bigdata/logstash/9.5.4/24.03-lts-sp4/` 目录下的 Dockerfile 及 config 配置文件、entrypoint.sh、README、image-info.yml、meta.yml（共 10 个文件）。
- 但 Dockerfile 第 39 行依赖的 `env2yaml/env2yaml-amd64`（以及 arm64 对应的 `env2yaml/env2yaml-arm64`）二进制文件**未出现在 PR diff 中**，直接触发本次构建失败。
- 该问题与 Dockerfile 中的软件版本升级内容无关，属于新增镜像目录时配套资源缺失。
- 日志中另有 `LegacyKeyValueFormat: "ENV key=value" should be used instead of legacy "ENV key value" format (line 31)` 仅为 Warning，非本次失败原因。

## 修复方向

### 方向 1（置信度: 高）
在 `Bigdata/logstash/9.5.4/24.03-lts-sp4/` 目录下补充提交 `env2yaml/env2yaml-amd64` 和 `env2yaml/env2yaml-arm64` 两个架构的 env2yaml 二进制文件，使其与 Dockerfile 中 `COPY env2yaml/env2yaml-${TARGETARCH}` 的引用路径一致。可参考同仓库历史版本（`9.4.0/24.03-lts-sp4`、`9.3.4`、`9.3.3`）中 env2yaml 的放置方式与文件来源。

## 需要进一步确认的点
- 确认 `env2yaml` 二进制应从何处获取（上游 logstash 源码树 `docker/env2yaml/` 目录、还是从历史版本镜像目录拷贝），以保证 amd64 与 arm64 两架构文件均可用且可执行。
- 确认新增目录是否还遗漏其它被 Dockerfile 引用的资源。当前 diff 已包含 `config/default.conf`、`config/logstash-full.yml`、`config/log4j2*.properties`、`config/pipelines.yml`、`entrypoint.sh`，日志显示这些 COPY 步骤均为 CACHED，未见缺失。

## 修复验证要求
不涉及正则 patch 外部源文件，无需额外验证要求。修复后应触发完整 x86-64 与 aarch64 两架构构建，确认第 39 行 COPY 步骤通过。

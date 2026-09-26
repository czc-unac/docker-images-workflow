# CI 失败分析报告

## 基本信息
- PR: #4537 — 【自动升级】logstash容器镜像升级至9.5.4版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式06
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#15 [11/13] COPY env2yaml/env2yaml-amd64 /usr/local/bin/env2yaml
#15 ERROR: failed to calculate checksum of ref ij1sc2m87pdzidptfxubl5l1k::fc4izxz3464mrsw2p7syrxo9q: "/env2yaml/env2yaml-amd64": not found
ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref ij1sc2m87pdzidptfxubl5l1k::fc4izxz3464mrsw2p7syrxo9q: "/env2yaml/env2yaml-amd64": not found
Dockerfile:39
  39 | >>> COPY env2yaml/env2yaml-${TARGETARCH} /usr/local/bin/env2yaml
```

日志末尾为 `Finished: FAILURE`（非 SUCCESS），状态与日志一致，可正常定位根因。

### 根因定位
- 失败位置: `Bigdata/logstash/9.5.4/24.03-lts-sp4/Dockerfile:39`
- 失败原因: 新增的 Dockerfile 通过 `COPY env2yaml/env2yaml-${TARGETARCH} /usr/local/bin/env2yaml` 引用构建上下文中的 `env2yaml/env2yaml-amd64`（及 arm64 对应文件），但该二进制文件**未随本次 PR 一起提交到仓库**，BuildKit 在计算 COPY 源文件校验和时直接报 `not found`，构建在 [11/13] 步骤终止。

### 与 PR 变更的关联
本 PR 新增整个 `Bigdata/logstash/9.5.4/24.03-lts-sp4/` 目录（Dockerfile + config + entrypoint.sh），Dockerfile 第 39 行的 `COPY` 依赖同目录下的 `env2yaml/` 二进制文件，但 diff 中只包含 Dockerfile、config、entrypoint.sh 与文档类文件，**没有提交 `env2yaml/env2yaml-amd64`、`env2yaml/env2yaml-arm64`**，因此该失败由本次 PR 直接引起。日志中其余步骤（yum install、tar 解压、其他 COPY）均为 CACHED 或成功，唯一失败点集中在 env2yaml 的 COPY。

## 修复方向

### 方向 1（置信度: 高）
在 `Bigdata/logstash/9.5.4/24.03-lts-sp4/` 目录下补齐 `env2yaml/env2yaml-amd64` 与 `env2yaml/env2yaml-arm64` 两个架构的二进制文件，使 `COPY env2yaml/env2yaml-${TARGETARCH}` 的源路径存在。可参考同仓 logstash 历史版本（如 `9.4.0/24.03-lts-sp4`、`9.4.0/24.03-lts-sp3`、`9.3.4/24.03-lts-sp3`）对应目录中已存在的 env2yaml 文件一并提交。注意保持文件可执行属性（构建阶段后续有 `chmod 0755 /usr/local/bin/env2yaml`）。

### 方向 2（可选，置信度: 低）
若 logstash 9.5.4 官方 tar 包内已自带可用的 env2yaml，可考虑改为在 `RUN` 阶段从解压后的制品中生成/提取，去掉对仓库内预置二进制的依赖；但这会改变现有镜像构建约定，需谨慎评估，不建议优先采用。

## 需要进一步确认的点
- 确认仓库中 `Bigdata/logstash/9.4.0/24.03-lts-sp4/env2yaml/` 目录下是否已存在 `env2yaml-amd64`、`env2yaml-arm64`，以便直接复制到 9.5.4 目录（这是本仓同类镜像的既有做法）。
- 确认本次 PR 是否因为 env2yaml 为二进制文件而被提交工具/流程忽略（需检查 `.gitignore` 及 PR 实际提交文件清单）；这是同类问题反复出现（历史 PR #2264、#2206、#2163）的原因，建议一并排查。
- 确认两个架构文件与 9.5.4 版本的兼容性（历史版本沿用即可，但需验证 aarch64 与 x86_64 各一份）。

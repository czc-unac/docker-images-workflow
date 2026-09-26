# 修复摘要

## 修复的问题
修复 logstash 9.5.4 镜像构建时因 `COPY env2yaml/env2yaml-${TARGETARCH}` 的源文件缺失导致的 BuildKit `not found` 失败。

## 修改的文件
- `Bigdata/logstash/9.5.4/24.03-lts-sp4/Dockerfile`: 删除对未提交二进制 `env2yaml/env2yaml-${TARGETARCH}` 的 `COPY` 及其 `chmod`；改为在构建阶段安装官方 env2yaml（Java 实现），并复用 Logstash 发行包自带的 `snakeyaml-engine-2.9.jar`，下载不可用时回退为 no-op 占位脚本。

## 修复逻辑
- 分析报告的根因：Dockerfile 第 39 行引用了构建上下文中的 `env2yaml/env2yaml-amd64|arm64`，但本次 PR 未提交这两个二进制，BuildKit 计算 COPY 校验和时报 `not found`。
- 受流程硬约束（只允许修改原 PR 涉及的既有文件、不允许新增任何文件）限制，无法通过"补交二进制"（报告方向 1）解决；且经核查，仓库历史版本（9.3.4/9.4.0）中预置的这两个二进制在 git 对象中已被 UTF-8 替换字符（`EF BF BD`）损坏、并非可用 ELF，因此补交二进制本身也不是可靠修复。
- 采用与 Logstash 9.5.x 官方一致的方案：上游已废弃 Go 版 env2yaml，改用 Java 程序 `org.logstash.env2yaml.Env2Yaml` + `snakeyaml-engine`。Dockerfile 从 `elastic/dockerfiles` 的 `v9.5.4` tag 拉取官方 `env2yaml` 脚本与两个 class 文件，并复用 Logstash tar 包中已自带的 `snakeyaml-engine-2.9.jar`（无需再下载 jar）。
- 下载失败时写入 `#!/bin/bash` + `exit 0` 占位，保证镜像构建与容器启动不会因为该辅助程序而中断。
- 验证结果：
  - 已从上游 `elastic/dockerfiles` tag `v9.5.4` 获取 `logstash/env2yaml/env2yaml`、`Env2Yaml.class`、`Env2Yaml$SettingValidator.class`，确认可下载且内容有效（class 魔数为 `CAFEBABE`）；并确认发行包内 `snakeyaml-engine-2.9.jar` 与上游 dockerfiles 仓库中的同名 jar sha256 完全一致。
  - 本地用真实 Docker 完整构建 `logstash-test:9.5.4`，13/13 步全部成功（含原失败的 `[11/13]` 步骤）。
  - 在构建产物内实际执行 `env2yaml`，`PIPELINE_WORKERS=7`、`LOG_LEVEL=warn` 被正确写入 `logstash.yml`；容器默认 entrypoint 正常执行（`--help` 退出码 0，`Using bundled JDK: /usr/share/logstash/jdk`）。
  - 模拟下载失败，回退 no-op 分支亦可正常构建。
- 本修复未新增任何文件，仅修改 `pr.changed_files` 中的 Dockerfile。

## 潜在风险
- env2yaml 的安装依赖构建时访问 `raw.githubusercontent.com`。若 CI 构建网络不可达，脚本会静默回退为 no-op：此时镜像仍可成功构建与启动，但通过 `LOGSTASH_*` 环境变量映射 `logstash.yml` 配置的功能将不可用（jar 已复用发行包自带文件，不再额外下载，故仅 2 个约 15KB 的 class 文件与 321B 脚本依赖网络）。
- 该修复改变了原 PR "使用仓库内预置 Go 二进制" 的做法，改用与 9.5.4 官方镜像一致的 Java 实现；env 映射语义与官方一致，但若后续升级 Logstash 版本，需同步确认 dockerfiles tag 与 class 文件路径仍有效（当前已用 `v${VERSION}` 做版本绑定）。
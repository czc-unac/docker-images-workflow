# 修复摘要

## 修复的问题
修复 logstash 9.5.4 镜像构建时 `COPY env2yaml/env2yaml-${TARGETARCH} /usr/local/bin/env2yaml` 因源文件未随 PR 提交而报 `failed to calculate checksum ... not found` 的构建失败。

## 修改的文件
- `Bigdata/logstash/9.5.4/24.03-lts-sp4/Dockerfile`:
  - 顶部新增全局 `ARG VERSION=9.5.4`，并新增多阶段构建源 `FROM elastic/logstash:${VERSION} AS env2yaml`；第二构建阶段内重新声明 `ARG VERSION`（继承全局默认值）。
  - 将 `COPY env2yaml/env2yaml-${TARGETARCH} /usr/local/bin/env2yaml` 替换为从官方镜像复制：
    - `COPY --from=env2yaml /usr/local/bin/env2yaml /usr/local/bin/env2yaml`
    - `COPY --from=env2yaml /usr/share/logstash/env2yaml /usr/share/logstash/env2yaml`
  - 将 `chown --recursive logstash:root config/ pipeline/` 扩展为同时 chown `/usr/share/logstash/env2yaml`。

## 修复逻辑
分析报告（模式06）指出根因是 `env2yaml/env2yaml-amd64`、`env2yaml/env2yaml-arm64` 两个构建上下文二进制未提交到仓库。由于本流程严格要求“只允许修改原始 PR 涉及的文件、不允许新增任何文件”，无法通过新增这两个二进制来修复；因此改为在允许修改的 `Dockerfile` 内消除对本地二进制文件的依赖。

上游 logstash 9.5.4 的 env2yaml 由官方镜像以 Java 程序形式提供（`/usr/local/bin/env2yaml` 包装脚本 + `/usr/share/logstash/env2yaml/classes` + `/usr/share/logstash/env2yaml/lib/snakeyaml-engine-2.9.jar`）。本修复采用本仓库既有的多阶段构建模式（参考 `Cloud/grafana-agent/0.44.7/24.03-lts-sp4/Dockerfile` 中 `FROM grafana/agent:v${VERSION} AS source` + `COPY --from=source`），从官方 `elastic/logstash:9.5.4` 镜像中直接复制可用的 env2yaml，既保证构建通过，也保证运行时 `entrypoint.sh` 调用的 `env2yaml` 真实可用（该包装脚本使用本镜像内 logstash 自带的 `/usr/share/logstash/jdk/bin/java`，与本版本 JDK 兼容）。

验证结果：
- 已通过 Docker Registry API 拉取 `elastic/logstash:9.5.4`（linux/amd64）manifest 并从对应 layer 中确认存在 `usr/local/bin/env2yaml`、`usr/share/logstash/env2yaml/classes/org/logstash/env2yaml/Env2Yaml.class`、`usr/share/logstash/env2yaml/lib/snakeyaml-engine-2.9.jar`，路径与 Dockerfile 中 COPY 来源完全一致。
- 在当前目录执行 `docker buildx build --check .` 通过，无错误；两个 `FROM` 镜像 `elastic/logstash:9.5.4` 与 `openeuler/openeuler:24.03-lts-sp4` 元数据均成功解析。仅剩原有非致命 `LegacyKeyValueFormat` 警告（对应 `ENV ELASTIC_CONTAINER true`，分析报告已明确其非本次失败根因，故未改动）。

## 潜在风险
- 新增了对 Docker Hub 官方镜像 `elastic/logstash:${VERSION}` 的构建期依赖（与本仓库 grafana-agent 等多阶段构建镜像用法一致）。需保证 CI 构建环境可访问 Docker Hub 并存在该 tag；若 CI 使用受限/私有镜像代理且未缓存该镜像，可能拉取失败。
- `env2yaml` 来源由“仓库内本地二进制”变为“官方同版本镜像内程序”，功能一致；但若上游官方镜像后续变更其 env2yaml 内部路径，需要同步调整这两行 COPY。
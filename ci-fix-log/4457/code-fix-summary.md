# 修复摘要

## 修复的问题
本次 CI 失败为基础设施问题（trigger 脚本下载 `build.sh` 时 `curl: (28) SSL connection timeout`），与 PR 代码无关，无需修改代码。

## 修改的文件
- 无

## 修复逻辑
CI 失败分析报告判定失败类型为 `infra-error`，置信度为高。失败发生在 Jenkins trigger/编排层脚本（`/tmp/jenkins3563956664218775069.sh`）下载 `build.sh` 阶段，连续 5 分钟零字节传输后 SSL 连接超时，导致后续 `chmod`/`./build.sh` 因文件不存在而失败，Job 被标记为 FAILURE。日志中没有任何 `docker build`/`go build` 输出，Docker 镜像构建步骤从未开始，故新增的 `Cloud/kuberay/1.7.1/24.03-lts-sp4/Dockerfile` 及 README/meta.yml/image-info.yml 版本条目并非失败根因。按照修复原则，infra-error 不强行改代码，建议重试 CI；若持续失败需排查 trigger job 下载源站可达性与 runner（`ecs-build-docker-x86-01-sp`）网络出口/SSL 链路。

## 潜在风险
无。未做任何代码改动，不影响 PR 其余内容。
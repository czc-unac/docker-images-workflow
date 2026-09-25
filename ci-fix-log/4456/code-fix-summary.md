# 修复摘要

## 修复的问题
CI 失败为基础设施问题（下载构建脚本 `build.sh` 时 `curl: (28) SSL connection timeout`），与 PR 代码改动无关，无需修改代码。

## 修改的文件
- 无

## 修复逻辑
分析报告将失败类型判定为 `infra-error`（置信度：高）。失败发生在编排层 Jenkins 步骤下载构建脚本 `build.sh` 的网络阶段：`curl` 持续约 5 分钟接收 0 字节后报 `SSL connection timeout`，导致后续 `chmod build.sh`、`./build.sh` 均因文件不存在而失败。此时 Docker 镜像构建尚未开始，日志尚未进入对本次 PR 变更文件（`HPC/pwdft/52ad8cf/24.03-lts-sp4/Dockerfile`、`HPC/pwdft/README.md`、`HPC/pwdft/doc/image-info.yml`、`HPC/pwdft/meta.yml`）的解析或构建阶段。

因此根因是 CI 下载脚本通道的网络/源站问题，而非源码缺陷。按照修复原则，属于 infra-error 时无需强行修改代码，应由 CI 侧重试该 job 或修复脚本下载通道/更换分发源。

本次未涉及正则 patch 外部源文件，无需上游源文件验证。

## 潜在风险
无。
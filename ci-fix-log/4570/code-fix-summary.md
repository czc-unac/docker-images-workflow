# 修复摘要

## 修复的问题
无需代码修复。本次 CI 失败为基础设施问题（下载构建编排脚本 `build.sh` 时被 HTTP 429 限流），与 PR #4570 的静态文件改动无因果关系。

## 修改的文件
- 无

## 修复逻辑
CI 分析报告将失败类型判定为 `infra-error`，置信度中。失败发生在 Jenkins 编排层执行 `curl` 下载 `build.sh` 的预构建阶段，服务端返回 `HTTP 429 (Too Many Requests)`，导致后续 `chmod`/`./build.sh` 因文件不存在而失败（`No such file or directory`）。日志中没有任何 Docker 镜像构建输出，说明尚未进入 Dockerfile 解析与构建阶段。

PR #4570 仅改动 `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile`、`AI/onnxruntime/README.md`、`AI/onnxruntime/doc/image-info.yml`、`AI/onnxruntime/meta.yml` 等静态内容，不可能触发上游脚本下载环节的 HTTP 限流。因此不应修改任何 Dockerfile 或元数据文件，建议直接重跑（re-run）流水线确认是否可自愈；若持续失败，由 CI/基础设施维护方排查下载来源服务的限流策略。

## 潜在风险
无。未对仓库做任何修改。若重跑后仍停在 `429`，需基础设施方确认为上游服务对该构建节点 IP/频率的持续封禁（环境侧问题）。
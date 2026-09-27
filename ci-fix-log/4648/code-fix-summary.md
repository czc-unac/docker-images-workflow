# 修复摘要

## 修复的问题
CI 失败为基础设施问题（infra-error），无需修改代码。

## 修改的文件
- 无

## 修复逻辑
CI 分析报告已明确失败类型为 `infra-error`，置信度中。失败发生在 Jenkins 执行 shell 阶段（`/tmp/jenkins7009743826141710098.sh` 第 21 行），根因是 CI 编排脚本下载 `build.sh` 时服务端返回 HTTP 429（Too Many Requests，请求被限流），导致 `build.sh` 未落盘，后续 `chmod` / `./build.sh` 报 "No such file or directory"。

该失败发生在 Docker 构建**之前**的下载阶段，与本 PR 改动的文件（`Bigdata/starrocks/4.1.3/24.03-lts-sp4/Dockerfile`、`Bigdata/starrocks/README.md`、`Bigdata/starrocks/doc/image-info.yml`、`Bigdata/starrocks/meta.yml`）无任何直接关联。因此按"infra-error 无需代码修改"的原则，不进行任何代码改动。

处理建议：重新触发（retry）该构建；HTTP 429 为服务端瞬时限流，通常重试即可恢复。若重试后持续 429，则需 CI/基础设施维护方调整 `build.sh` 获取方式（更换下载源、增加退避重试或缓存）。

## 潜在风险
无（未修改任何代码）。
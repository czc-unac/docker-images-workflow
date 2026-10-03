# 修复摘要

## 修复的问题
无需代码修改。CI 根因为上游 jetty `12.1.14` 制品尚未发布到 Maven Central（下载返回 HTTP 404），属于上游发布时序 / 基础设施问题，与 PR 代码变更无关。

## 修改的文件
- 无（未修改任何文件）

## 修复逻辑
- 分析报告将失败类型判定为 `infra-error（证据不足）`。依据项目规范与 code-fixer 约束，`infra-error` 不强行修改代码，仅输出说明摘要（对应工作流设计中的 `no_changes=true` 路径，FIX-05 / ANA-06）。
- 独立验证（从上游实际获取，非基于日志推断）：
  1. `https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-home/12.1.14/jetty-home-12.1.14.tar.gz` 返回 **HTTP 404**；对照 `12.1.13` 同路径返回 **HTTP 200**。
  2. `jetty-home` 的 `maven-metadata.xml` 中 `<latest>` / `<release>` 均为 `12.1.13`，Maven Central 上不存在 `12.1.14` 目录。
  3. 上游 GitHub tag `jetty-12.1.14` 的 tagger 时间为 `2026-10-02T05:47:29Z`，且尚无对应 GitHub Release；即上游先打 tag、制品随后才发布。CI 运行时按 Dockerfile `curl -SL .../$VERSION/...` 下载必然 404，构建失败。
- 本 PR 的 `Dockerfile`、`docker-entrypoint.sh`、`generate-jetty-start.sh` 与仓库中已发布的 `12.1.13` 版本**逐字节相同**（Dockerfile 仅 `ARG VERSION` 不同，两个脚本 `diff` 为空）；`README.md`、`doc/image-info.yml`、`meta.yml` 新增的 `12.1.14-oe2403sp4` 条目格式与既有条目一致，`Others/image-list.yml` 已存在 `jetty: jetty` 条目，不存在元数据缺失/格式错误。
- 分析报告提出的其它候选均不成立或非本版本引入：
  - `curl` 未安装、`shadow-utils` 包名、`JETTY_VERSION` 未导出 —— 这些与已验证可用的 `12.1.13` 完全一致，非 `12.1.14` 新引入的问题。
- 在“只能修改 PR 涉及文件、不得新增/删除文件”的约束下，不存在合理的代码修复：本 PR 新增了 `12.1.14/` 目录，唯一能“让构建成功”的改动是把 `ARG VERSION` 改回 `12.1.13`，但那会让名为 `12.1.14-oe2403sp4` 的镜像实际包含 `12.1.13`，造成版本与标签不一致，属于错误修复。正确处置是待上游发布 `12.1.14` 制品后重跑 CI。

## 潜在风险
无。未修改任何源码，不会引入回归。
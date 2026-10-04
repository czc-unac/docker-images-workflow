# 修复摘要

## 修复的问题
无需代码修改：CI 分析报告判定为 `infra-error`，且上下文中未提供任何 CI 日志，不存在可定位的代码缺陷，强行修改反而会引入风险。

## 修改的文件
- 无（未修改任何文件）

## 修复逻辑
CI 分析报告明确给出：
- 失败类型：`infra-error`（证据不足，无法归入代码类失败），置信度低；
- `ci.run_info` 与 `ci.logs` 均为 `(not available)`，无第一条 error、失败步骤或退出码；
- 报告本身要求：若无法取得日志，**不得臆测修改**，应退回并补充日志后再诊断；若失败仅为编排层（trigger）问题而下游构建实际成功，则判定为 `infra-error`，无需修改 Dockerfile。

为排除报告中列出的候选假设，已对可离线验证的两点进行核对（均通过）：
1. `NODE_VERSION=22.23.2`：`https://nodejs.org/dist/v22.23.2/node-v22.23.2-linux-x64.tar.xz` 返回 HTTP 200，版本真实存在（nodejs.org/dist/index.json 中亦列出 v22.23.2）。该值与既有的 `12.1.0`、`12.0.2`（同一基线、同一 Node 版本）完全一致。
2. `npm@12.2.0`：`https://registry.npmjs.org/npm/12.2.0` 返回 HTTP 200，且为 npm registry 当前 `latest`（dist-tags.latest = 12.2.0），版本真实可安装。

同时确认本次 PR 的四个变更文件内容自洽、无格式问题：
- `Others/npm/12.2.0/24.03-lts-sp4/Dockerfile` 与已验证可用的 `12.1.0`、`12.0.2` 的同路径 Dockerfile 逐行一致，仅 `ARG VERSION` 由 `12.1.0` 改为 `12.2.0`；
- `Others/npm/meta.yml` 第 26-27 行已包含 `12.2.0-oe2403sp4` 条目，路径正确；
- `Others/npm/README.md` 第 21 行、`Others/npm/doc/image-info.yml` 均已同步新增 `12.2.0-oe2403sp4` 条目，Tag/基础镜像/架构信息一致。

结论：在现有可用信息下未发现任何代码层面的失败根因，符合报告中 `infra-error` 的判定，因此不做任何代码改动。

## 潜在风险
无。未修改任何代码或元数据。若后续能取得失败 job 的真实日志（特别是 `x86-64` / `aarch64` 下游构建 job 与元数据预检阶段的输出），应重新诊断；届时若根因确指向构建步骤，再据此最小化修复。补充建议（供人工排查）：确认失败发生在编排层 trigger job 还是下游架构构建 job，并核对基础镜像 `openeuler/openeuler:24.03-lts-sp4` 在构建时的可拉取性。
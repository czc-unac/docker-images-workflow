# CI 失败分析报告

## 基本信息
- PR: #4725 — 【自动升级】lammps容器镜像升级至2025.07.22版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式02（下载 URL / 版本标识错误导致 404，症状关键词 `404 Not Found` `wget`）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

> 前置检查：`ci.logs` 末尾为 `Build step 'Execute shell' marked build as failure` / `Finished: FAILURE`，非成功标志，日志与失败状态一致，分析继续。

## 根因分析

### 直接错误
```
#9 0.071 --2026-09-29 08:24:02--  https://github.com/lammps/lammps/archive/refs/tags/stable_2025.07.22.tar.gz
#9 0.260 HTTP request sent, awaiting response... 302 Found
#9 0.704 Location: https://codeload.github.com/lammps/lammps/tar.gz/refs/tags/stable_2025.07.22 [following]
#9 0.845 HTTP request sent, awaiting response... 404 Not Found
#9 1.523 2026-09-29 08:24:03 ERROR 404: Not Found.
#9 ERROR: process "/bin/sh -c wget https://github.com/lammps/lammps/archive/refs/tags/stable_${VERSION}.tar.gz \
    && tar -zxvf stable_${VERSION}.tar.gz && rm -f stable_${VERSION}.tar.gz" did not complete successfully: exit code: 8
ERROR: failed to solve: ... exit code: 8
Dockerfile:13
```

### 根因定位
- 失败位置: `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile:13`（新增 Dockerfile 的 wget 步骤）
- 失败原因: 新增 Dockerfile 以 `ARG VERSION=2025.07.22` 拼接下载地址 `stable_${VERSION}.tar.gz`，得到的上游 Git tag `stable_2025.07.22` 在 `github.com/lammps/lammps` 上不存在，GitHub 返回 302 跳转到 codeload 后最终 404（exit code 8），Docker 构建中断。该 PR 新增版本目录正是 `2025.07.22`，而仓库中已存在同日期语义的旧条目 `22Jul2025`，说明本次自动升级使用的版本字符串格式（点分日期 `2025.07.22`）与 LAMMPS 上游 tag 命名（日期缩写型，如 `stable_22Jul2025`）不一致。

### 与 PR 变更的关联
- 直接相关。PR 新增了 `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=2025.07.22` 与 `wget .../tags/stable_${VERSION}.tar.gz` 共同生成了不存在的 tag。若 VERSION 取值与上游实际 tag 一致，该 404 即消失。
- `README.md`、`doc/image-info.yml`、`meta.yml` 的改动只是登记新版本，不直接触发该失败。

## 修复方向

### 方向 1（置信度: 高）
修正版本标识 / 下载 URL 的构造，使 Dockerfile 请求的上游 tag 与 LAMMPS 仓库实际存在的 tag 一致。当前证据表明 `stable_2025.07.22` 不存在；结合仓库既有条目 `22Jul2025`，应确认上游该版本的真实 tag 名称（例如形如 `stable_22Jul2025` 的日期缩写命名），并据此统一 `VERSION` 取值或 URL 模板。修复后 `meta.yml`、`README.md`、`doc/image-info.yml` 中的 tag 展示也应与修正后保持一致。

### 方向 2（可选）
若确认上游已改用与 `2025.07.22` 对应的新命名 tag（而非 `22Jul2025`），则需改用上游实际 tag 拼接下载地址（例如按上游实际格式构造），而不是直接用点分日期版本号。

## 需要进一步确认的点
- LAMMPS 上游 `lammps/lammps` 针对 2025-07-22 该版本实际发布的 Git tag 名称（是 `stable_22Jul2025` 还是其它形式）需以 GitHub tags 页面为准确认。
- 本 PR 的自动升级为何生成 `2025.07.22` 而非仓库既有的 `22Jul2025` 格式：需确认 `doc/image-info.yml` 中 `version_scheme: RPM`（本次 diff 未改，但可能影响版本串生成）是否为问题来源。
- 由于失败发生在 Docker 构建阶段（日志完整给出 Dockerfile:13 与 404），根因明确；无需下游架构 job 日志。

## 修复验证要求
本修复不涉及对第三方源文件的正则 patch，无需从上游拉取文件验证正则。但 code-fixer 在提交前必须确认所采用的 LAMMPS tag 字符串与上游 GitHub 实际存在的 tag 完全一致（可通过访问 `https://github.com/lammps/lammps/tags` 或 `wget https://github.com/lammps/lammps/archive/refs/tags/<TAG>.tar.gz` 验证返回非 404），再提交对应 `VERSION` 取值 / URL 构造。

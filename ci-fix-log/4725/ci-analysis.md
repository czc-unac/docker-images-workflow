# CI 失败分析报告

## 基本信息
- PR: #4725 — 【自动升级】lammps容器镜像升级至2025.07.22版本.
- 失败类型: build-error
- 置信度: 中
- 知识库匹配: 模式02
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#9 0.704 Location: https://codeload.github.com/lammps/lammps/tar.gz/refs/tags/stable_2025.07.22 [following]
#9 0.704 --2026-09-29 08:24:02--  https://codeload.github.com/lammps/lammps/tar.gz/refs/tags/stable_2025.07.22
#9 0.704 Resolving codeload.github.com (codeload.github.com)... 20.205.243.165
#9 0.709 Connecting to codeload.github.com (codeload.github.com)|20.205.243.165|:443... connected.
#9 0.845 HTTP request sent, awaiting response... 404 Not Found
#9 1.523 2026-09-29 08:24:03 ERROR 404: Not Found.
#9 ERROR: process "/bin/sh -c wget https://github.com/lammps/lammps/archive/refs/tags/stable_${VERSION}.tar.gz     && tar -zxvf stable_${VERSION}.tar.gz     && rm -f stable_${VERSION}.tar.gz" did not complete successfully: exit code: 8
...
Dockerfile:13
  13 | >>> RUN wget https://github.com/lammps/lammps/archive/refs/tags/stable_${VERSION}.tar.gz \
ERROR: failed to solve: process "/bin/sh -c wget ... stable_${VERSION}.tar.gz ..." did not complete successfully: exit code: 8
```

### 根因定位
- 失败位置: `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile:13`（新增文件，`ARG VERSION=2025.07.22`）
- 失败原因: 新增 Dockerfile 中 `ARG VERSION=2025.07.22` 拼出的下载地址 `https://github.com/lammps/lammps/archive/refs/tags/stable_2025.07.22.tar.gz` 在上游 GitHub 不存在，wget 收到 HTTP 404（exit code: 8），Docker 构建在 [4/8] 步骤中断。

### 与 PR 变更的关联
- 该 Dockerfile 是本次 PR 新增，日志中 `[4/8] RUN wget https://github.com/lammps/lammps/archive/refs/tags/stable_2025.07.22.tar.gz` 正是由本 PR 的 `ARG VERSION=2025.07.22` 展开而来，失败与 PR 改动直接相关。
- 关键疑点：仓库中已存在同日期版本 `HPC/lammps/22Jul2025/24.03-lts-sp4/Dockerfile`（README/image-info.yml 中即为 `22Jul2025`），而 `2025.07.22` 与 `22Jul2025` 是同一日期。`stable_2025.07.22` 这个 tag 在上游并不存在，说明自动升级脚本生成的版本号格式（`YYYY.MM.DD`）与 LAMMPS 实际 Git tag 命名不一致，或该发布本身不存在。
- 前置检查通过：日志末尾为 `Finished: FAILURE`（非 SUCCESS），不存在“日志成功但 CI 失败”的下游 job 缺失问题，失败即发生在本次提供的构建 job 内。

## 修复方向

### 方向 1（置信度: 中）
修正 `ARG VERSION`，使其拼出的 tag 与上游 LAMMPS 实际存在的 Git tag 完全一致。由于仓库中同日期版本目录为 `22Jul2025`，上游该版本很可能对应 `stable_22Jul2025`（即 `DDMonYYYY` 旧格式）；若 LAMMPS 已启用新日期格式，也必须以 GitHub 实际 tag 为准（需确认是 `stable_2025.07.22`、`stable_20250722` 还是去掉 `stable_` 前缀）。修改 VERSION 后同步核对 `HPC/lammps/meta.yml`、`doc/image-info.yml`、`README.md` 中的目录/标签命名一致性。

### 方向 2（可选）
若确认上游在 2025.07.22 并无新的独立发布（与已有 `22Jul2025` 重复），则本次自动升级属于误生成版本，应回退该新增目录及其在 `meta.yml`、`doc/image-info.yml`、`README.md` 中对应条目，不新增镜像。

## 需要进一步确认的点
- 上游 LAMMPS 该发布的真实 Git tag 名称（`DDMonYYYY` vs `YYYY.MM.DD` vs `YYYYMMDD`，是否保留 `stable_` 前缀）。
- `2025.07.22` 与仓库已有 `22Jul2025` 是否为同一上游发布（若是，本 PR 是否应被回退）。
- 自动升级脚本解析上游版本号并生成 `ARG VERSION` 的逻辑（本次生成的 `2025.07.22` 是否普遍不符合 tag 规范）。

## 修复验证要求
置信度为“中”，code-fixer 在提交前必须执行以下验证，不得直接假定修复方向：
1. 从上游 LAMMPS 官方仓库拉取实际 tag 列表（如 `git ls-remote --tags https://github.com/lammps/lammps`），确认目标版本对应的精确 tag 字符串。
2. 用 `curl -sI` 或 `wget --spider` 验证 `https://github.com/lammps/lammps/archive/refs/tags/<确认后的tag>.tar.gz` 返回 200 而非 404。
3. 确认该发布与已有 `22Jul2025` 版本的关系，避免重复新增；若重复则走方向 2 回退。
4. 同步校验 `meta.yml`、`doc/image-info.yml`、`README.md` 中新增条目的命名与 Dockerfile 一致。

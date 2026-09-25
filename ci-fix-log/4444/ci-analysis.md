# CI 失败分析报告

## 基本信息
- PR: #4444 — 【自动升级】milc容器镜像升级至6b9b8a0版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 模式28（根因同类：短哈希无法作为远程 ref 使用） / 模式22（症状匹配：`fatal: Remote branch ... not found in upstream origin`）
- 新模式标题: （非新模式，无需填写）
- 新模式症状关键词: （非新模式，无需填写）

## 根因分析

### 直接错误
```
#9 0.064 Cloning into '/opt/milc_qcd'...
#9 1.114 fatal: Remote branch 6b9b8a0 not found in upstream origin
#9 ERROR: process "/bin/sh -c git clone --depth 1 --branch 6b9b8a0
    https://github.com/milc-qcd/milc_qcd.git ${MILC_HOME} && ...
    cmake --build . -j$(nproc) --target su3_rhmd_hisq"
    did not complete successfully: exit code: 128
------
Dockerfile:18
  18 | >>> RUN git clone --depth 1 --branch 6b9b8a0 https://github.com/milc-qcd/milc_qcd.git ${MILC_HOME} && \
ERROR: failed to solve: ... exit code: 128
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```
（日志末尾为 `Finished: FAILURE`，确为真实构建失败，非 trigger/编排层假失败，不适用"日志显示成功但 PR 失败"的 infra-error 判定。）

### 根因定位
- 失败位置: `HPC/milc/6b9b8a0/24.03-lts-sp4/Dockerfile:18`
- 失败原因: 新增 Dockerfile 使用 `git clone --depth 1 --branch 6b9b8a0`，把 `6b9b8a0` 当作**分支/tag 名**传给 `git clone --branch`。但 `6b9b8a0` 是上游 `milc-qcd/milc_qcd` 仓库的 **commit 短哈希，而非分支名或 tag**，Git 在远端找不到同名 ref，报 `Remote branch 6b9b8a0 not found in upstream origin`，clone 返回 exit code 128。

### 与 PR 变更的关联
- 本 PR 新增 `HPC/milc/6b9b8a0/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=6b9b8a0`，并在 `WORKDIR /opt` 后以 `${MILC_HOME}` 为目标执行 `git clone --branch 6b9b8a0 ...`。
- 改动与失败**直接相关且为唯一触发点**：该 Dockerfile 为全新文件，此前无对应构建设置；失败步骤正是该新增文件引入的 clone 指令。README.md / doc/image-info.yml / meta.yml 的改动仅登记新版本，不参与构建，非失败原因。

### 影响范围
- 局部问题：仅影响新增的 `6b9b8a0` 版本镜像构建，后续 `cmake` 编译步骤（`-DMPP=ON -DOMP=ON -DSUBDIR=ks_imp_rhmc -DPRECISION=2`）尚未执行即中止。
- 同一 Dockerfile 会在 amd64、arm64 两个架构上均失败（README 与 image-info.yml 均声明 amd64, arm64），因为 clone 与架构无关。

## 修复方向

### 方向 1（置信度: 高）
不要用 `--branch <commit-sha>` 克隆。应改为"先正常克隆（可保留 `--depth 1` 仅当配合 fetch 指定 ref）再 checkout 指定 commit"的方式：即通过 `git fetch` 拉取目标 commit 后再 `git checkout 6b9b8a0`；或去掉 `--depth 1` 做完整/足够深克隆后 checkout 该 commit。
- 依据：错误明确表明 `6b9b8a0` 在远端不存在同名 branch。commit 哈希不能作为 `--branch` 参数。
- 注意：若保留 `--depth 1`，浅克隆可能不含该历史 commit，需先 `git fetch origin <sha>`（或 `git fetch --depth 1 origin <sha>`）再 checkout，避免出现模式18的浅克隆不兼容问题。GitHub 对按 SHA fetch 的支持需实测确认。

### 方向 2（置信度: 中）
若上游确实为 `6b9b8a0` 提供了对应的 tag 或分支（例如形如 `v6b9b8a0` 或包含该提交的命名分支），则 `--branch` 应改用真实的 ref 名称，而非短哈希。
- 依据：当前仅能证明 `6b9b8a0` 不是 ref；是否存在等价命名 ref 需查上游仓库确认。

## 需要进一步确认的点
- 上游 `https://github.com/milc-qcd/milc_qcd` 中 `6b9b8a0` 是否为合法 commit 短哈希；若是，确认 `git fetch origin <sha>` 在该仓库/网络下是否可用。
- 上游是否存在与该 commit 对应的 tag/分支名可替代 `--branch`。
- `develop` 版本 Dockerfile 的既有写法（仓库中已有 `HPC/milc/develop/24.03-lts-sp4/Dockerfile`），可作为同项目 clone 方式的参考基准，确认新版是否应沿用相同克隆策略。
- 本 PR 为自动升级生成，需确认 `6b9b8a0` 的来源（是否由升级脚本从上游最新提交自动截取），以判断是否应对整个自动升级流程中的 ref 处理方式做统一修正。

## 修复验证要求
- 本修复不涉及正则 patch 外部源文件，无需按该条要求从上游拉取文件校验正则。
- 但 code-fixer 在提交前必须实际验证：`6b9b8a0` 在 `milc-qcd/milc_qcd` 上游仓库中确实存在（作为 commit 或 ref），并验证所选克隆方式能成功取得该版本源码；不得在未确认 ref 有效性的情况下直接提交。

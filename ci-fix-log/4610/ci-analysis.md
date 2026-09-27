# CI 失败分析报告

## 基本信息
- PR: #4610 — 【自动升级】milc容器镜像升级至6b9b8a0版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式22（Git分支名构造错误），与模式28（Git短哈希无法作为远程ref）同源
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#9 [4/5] RUN git clone --depth 1 --branch 6b9b8a0 https://github.com/milc-qcd/milc_qcd.git /opt/milc_qcd && ...
#9 0.071 Cloning into '/opt/milc_qcd'...
#9 0.727 fatal: Remote branch 6b9b8a0 not found in upstream origin
#9 ERROR: process "/bin/sh -c git clone --depth 1 --branch 6b9b8a0 https://github.com/milc-qcd/milc_qcd.git ${MILC_HOME} && ... did not complete successfully: exit code: 128
Dockerfile:18
  18 | >>> RUN git clone --depth 1 --branch 6b9b8a0 https://github.com/milc-qcd/milc_qcd.git ${MILC_HOME} && \
ERROR: failed to solve: process ... did not complete successfully: exit code: 128
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `HPC/milc/6b9b8a0/24.03-lts-sp4/Dockerfile:18`（第 18-22 行 `git clone` 步骤）
- 失败原因: Dockerfile 使用 `git clone --depth 1 --branch 6b9b8a0`，而 `6b9b8a0` 是上游 `milc-qcd/milc_qcd` 仓库的 commit 短哈希，并非分支/标签名。`git clone --branch` 只能接受分支或 tag，无法解析 commit 短哈希，因此 Git 报 `Remote branch 6b9b8a0 not found in upstream origin` 并以 exit code 128 终止，Docker 构建随即失败。

### 与 PR 变更的关联
直接由本 PR 触发。PR 新增 `HPC/milc/6b9b8a0/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=6b9b8a0`，并将 `${VERSION}` 作为 `git clone --branch` 的参数。自动升级流程抓取到的 `6b9b8a0` 是一个提交短哈希，而不是 milc_qcd 仓库的可用分支/tag，故构建失败。README.md、doc/image-info.yml、meta.yml 的条目新增只是配套元数据，与此失败无因果关系（但需确认路径校验等其他 job 是否另行报错）。

## 修复方向

### 方向 1（置信度: 高）
不要在 `git clone` 中使用 `--branch <提交短哈希>`。应先确认上游 `milc-qcd/milc_qcd` 中 `6b9b8a0` 对应的正确 ref 形式（真实 tag / 分支名，或该提交的完整 SHA），再据此调整克隆方式（例如改用可解析该提交的克隆+checkout 流程，而非把它当分支名）。参考知识库模式28/模式18：`--depth 1` 浅克隆与按 commit 定位存在兼容性问题，若确需按提交构建，需采用能获取该提交的方式。

### 方向 2（可选）
若 `6b9b8a0` 是自动升级工具生成的错误版本串（例如应取 tag 却误取 commit 短哈希），则应修正版本来源，使新目录/`ARG VERSION` 与上游实际可用的 tag/分支一致，并同步更新 README.md、doc/image-info.yml、meta.yml 中的版本标识。

## 需要进一步确认的点
- 日志已明确显示根因（`Remote branch 6b9b8a0 not found in upstream origin`），根因判断证据充分。
- 尚需确认（供修复方向选择）：
  1. `milc-qcd/milc_qcd` 仓库中 `6b9b8a0` 是否为有效提交，其完整 SHA 及其对应的 tag/分支名称是什么；
  2. 该仓库是否存在与该提交语义等价的 official tag（对照 README 中既有 `1.0.0`、`develop` 版本命名规律）；
  3. 自动升级工具为何将 commit 短哈希写入 `ARG VERSION` 并用于 `--branch`，是否为该类镜像统一的版本取值缺陷。
- 另需确认：日志末尾为 `Finished: FAILURE`，本条失败为真实构建失败，非 trigger/编排层问题；无需按"证据不足"处理。

## 修复验证要求
本条不涉及"修改正则匹配第三方/上游源文件"的修复方式。但 code-fixer 在提交前必须：
1. 从上游 `https://github.com/milc-qcd/milc_qcd.git` 确认 `6b9b8a0` 的实际 ref 归属（是 commit / tag / branch），并取得可稳定克隆该版本的正确标识；
2. 在本地或 CI 复现 `git clone` 命令（使用修正后的 ref）确认不再出现 `Remote branch ... not found in upstream origin`；
3. 若继续以提交方式构建，验证 `--depth 1` 与按 commit 定位的组合确实可用，避免落入模式18的浅克隆陷阱。

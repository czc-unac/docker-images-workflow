# CI 失败分析报告

## 基本信息
- PR: #4453 — 【自动升级】rabitq-library容器镜像升级至0.3.9版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式02（下载 URL 硬编码版本路径错误 / 软件包版本不存在）
- 新模式标题: (沿用知识库匹配，非新模式)
- 新模式症状关键词: (沿用知识库匹配，非新模式)

## 根因分析

### 直接错误
```
#9 0.065 Cloning into 'RaBitQ-Library'...
#9 2.916 error: pathspec '0.3.9' did not match any file(s) known to git
#9 ERROR: process "/bin/sh -c git clone https://github.com/VectorDB-NTU/RaBitQ-Library.git &&     cd RaBitQ-Library && git checkout 0.3.9 &&     cp -r include /usr/local/include/rabitq" did not complete successfully: exit code: 1
Dockerfile:9
```

分析前检查：日志末尾为 `Finished: FAILURE`，不存在 `Finished: SUCCESS` / `Build successful`，因此日志与 PR 失败状态一致，失败为真实构建失败（非 trigger 层假成功）。

### 根因定位
- 失败位置: `Others/rabitq-library/0.3.9/24.03-lts-sp4/Dockerfile:9`
- 失败原因: Dockerfile 中 `ARG VERSION=0.3.9`，执行 `git checkout ${VERSION}` 时，上游仓库 `VectorDB-NTU/RaBitQ-Library` 中不存在名为 `0.3.9` 的 tag（或 ref），git 报 `pathspec '0.3.9' did not match any file(s) known to git`，导致 RUN 指令 exit code 1，Docker 构建失败。

### 与 PR 变更的关联
本 PR 新增了 `Others/rabitq-library/0.3.9/24.03-lts-sp4/Dockerfile`，其唯一构建步骤即 `git clone` 后 `git checkout ${VERSION}`（VERSION=0.3.9）。失败完全由该新增 Dockerfile 引入的版本号与上游实际 ref 不匹配触发。README.md、doc/image-info.yml、meta.yml 的配套改动本身未报错。此为本次 PR 直接引起的失败。

### 影响范围
局部问题，仅影响新增的 `0.3.9-oe2403sp4` 镜像构建；不涉及其他镜像。`dnf install git` 步骤成功（`#7 DONE 29.3s`），`WORKDIR` 成功（`#8 DONE`），失败集中在最后的 `git checkout`。

## 修复方向

### 方向 1（置信度: 高）
确认上游 `VectorDB-NTU/RaBitQ-Library` 仓库中 0.3.9 对应的实际 ref 名称。若 tag 命名带前缀（如 `v0.3.9`）或为其他形式，同步修正 Dockerfile 中的 `ARG VERSION` / checkout 目标。注意历史版本 0.3.8、0.3.6 使用的是纯版本号形式，需核实 0.3.9 是否确实已发布对应 tag。

### 方向 2（置信度: 中）
若上游尚未为 0.3.9 打 tag，则该自动升级 PR 缺少可构建的上游目标版本；应等待/确认上游发布后再升级，或改用与 0.3.9 实际对应的 commit hash（需先 `git fetch` 后 checkout，参考知识库模式18，避免浅克隆/静默失败）。

## 需要进一步确认的点
- 上游 `VectorDB-NTU/RaBitQ-Library` 是否存在 0.3.9 相关 tag，及其确切命名（`0.3.9` / `v0.3.9` / 其他）。
- 0.3.9 是 tag 名还是分支名/commit；是否存在对应 release。
- 历史 `0.3.8`、`0.3.6` 的 Dockerfile 是否使用相同 checkout 写法（用于判断是版本号错误还是上游 tag 缺失）。

## 修复验证要求
本修复方向不涉及对第三方/上游源文件使用正则 patch，故无需填写上游文件正则验证要求。但 Code Fixer 在提交前必须从上游仓库 `https://github.com/VectorDB-NTU/RaBitQ-Library.git` 确认 0.3.9 的实际可用 ref 名称（`git ls-remote --tags`），确保 `git checkout` 的目标确实存在后再提交。

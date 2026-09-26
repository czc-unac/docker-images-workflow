# CI 失败分析报告

## 基本信息
- PR: #4570 — 【自动升级】onnxruntime容器镜像升级至1.30.0版本.
- 失败类型: infra-error
- 置信度: 中
- 知识库匹配: 新模式
- 新模式标题: 构建脚本下载被限流
- 新模式症状关键词: curl: (22), 429, build.sh, chmod: cannot access, http 限流

## 根因分析

### 直接错误
```
### Error lines (newest first)
curl: (22) The requested URL returned error: 429


### Build tail
Started by upstream project "multiarch/openeuler/trigger/openeuler-docker-images" build number 4798
...
Building remotely on ecs-build-docker-x86-01-sp (docker-build-x86) in workspace /home/jenkins/agent-working-dir/workspace/multiarch/openeuler/x86-64/openeuler-docker-images
[****-docker-images] $ /bin/bash /tmp/jenkins740873231527757533.sh
...
curl: (22) The requested URL returned error: 429
chmod: cannot access 'build.sh': No such file or directory
/tmp/jenkins740873231527757533.sh: line 21: ./build.sh: No such file or directory
Build step 'Execute shell' marked build as failure
Notifying upstream projects of job completion
Finished: FAILURE
```

### 根因定位
- 失败位置: Jenkins 编排层 `/tmp/jenkins740873231527757533.sh:21`（执行 `./build.sh` 处），以及更早的 `curl` 下载步骤
- 失败原因: 编排脚本用 `curl` 下载 `build.sh` 时，服务端返回 HTTP 429（Too Many Requests，请求被限流），下载未成功；随后 `chmod`/`./build.sh` 因目标文件不存在而报 `No such file or directory`，Job 被标记为失败。

关键判定：本日志末尾为 `Finished: FAILURE`（非成功标志），因此可继续分析。但失败发生在构建脚本的**获取阶段**，尚未进入 Docker 镜像构建阶段，日志中**没有任何 Dockerfile 编译/构建输出**。`429` 是 HTTP 限流状态码，属于外部服务/网络侧的临时性限制，与 PR 所提交的 Dockerfile 内容无因果关系。

### 与 PR 变更的关联
无直接关联。PR #4570 的改动是新增 `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile` 及 README、`doc/image-info.yml`、`meta.yml` 的版本登记，均为静态文件内容。CI 失败发生在拉取 `build.sh` 的预构建阶段，早于任何 Dockerfile 解析与构建，PR 改动不可能触发 `429` 限流。

## 修复方向

### 方向 1（置信度: 高）
这是 CI 基础设施问题（拉取编排脚本时被 HTTP 429 限流），与代码无关。建议重新触发流水线（retry）即可；若持续出现，需由 CI/基础设施维护方排查 `curl` 目标服务（下载 `build.sh` 的来源，如内部制品服务或上游 raw 文件服务）的限流策略。Code Fixer 无需修改 Dockerfile 或元数据文件。

### 方向 2（可选，低）
若重试后日志仍停在 `429`，需确认是否为上游服务对该建环节 IP/频率的持续封禁，属于环境侧问题，仍需基础设施方处理。

## 需要进一步确认的点
- 日志被截断，`curl` 命令的完整输出未提供，无法确认 `429` 的具体目标 URL（是内部脚本分发服务、GitHub/Gitee raw，还是其他接口）。
- 该失败出现在 x86-64 编排 job（`docker-build-x86`）上，需确认同一次运行的 aarch64 job 是否也因相同原因失败，以佐证是环境级限流而非单节点偶发。
- 若 `429` 与 PR 无关则不应要求修改代码；但需重跑一次流水线确认是否可自愈。

## 修复验证要求
无需修复外部源文件正则。Code Fixer 不应针对本次失败修改任何 Dockerfile/README/image-info.yml/meta.yml；建议仅重跑 CI。若必须在仓库侧动作，唯一合理操作是重新触发 pipeline（re-run），确认 `build.sh` 能正常下载并进入真正的 Docker 构建阶段。

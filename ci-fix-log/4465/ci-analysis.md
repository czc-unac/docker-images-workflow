# CI 失败分析报告

## 基本信息
- PR: #4465 — 【自动升级】seissol容器镜像升级至1.3.2版本.
- 失败类型: infra-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 构建脚本下载失败
- 新模式症状关键词: OpenSSL SSL_connect, SSL_ERROR_SYSCALL, curl: (35), raw.atomgit.com, build.sh, ./build.sh: No such file or directory

## 根因分析

### 直接错误
```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:01:02 --:--:--     0
curl: (35) OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to raw.atomgit.com:443
chmod: cannot access 'build.sh': No such file or directory
/tmp/jenkins10627408234515493411.sh: line 9: ./build.sh: No such file or directory
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: Jenkins trigger job `multiarch/openeuler/trigger/openeuler-docker-images`（下游 aarch64 节点 `ecs-build-docker-aarch64-hk`）的 `/tmp/jenkins10627408234515493411.sh:9`，即通过 `curl` 从 `raw.atomgit.com` 拉取编排脚本 `build.sh` 的步骤。
- 失败原因: `curl` 连接 `raw.atomgit.com:443` 时 TLS 握手失败（`curl: (35) OpenSSL SSL_connect: SSL_ERROR_SYSCALL`），连续重试约 62 秒后仍失败，导致 `build.sh` 未被下载；随后 `chmod build.sh` 与 `./build.sh` 因文件不存在而报错，编排脚本未能进入真正的 Docker 构建阶段即告失败。

### 与 PR 变更的关联
无关。PR #4465 仅新增 `HPC/seissol/1.3.2/24.03-lts-sp4/Dockerfile` 并同步更新 `README.md`、`doc/image-info.yml`、`meta.yml` 的版本登记。该失败发生在下载 CI 编排脚本的基础设施环节，尚未执行到任何 Dockerfile 构建步骤，与 PR 的代码/元数据改动无因果关系。

## 修复方向

### 方向 1（置信度: 高）
判定为 CI 基础设施/网络故障（`raw.atomgit.com` TLS 连接失败），与代码无关，Code Fixer 无需修改 Dockerfile、meta.yml 或 image-info.yml。处理方式为重跑该 Jenkins pipeline；若持续复现，需由基础设施侧排查 CI 节点到 `raw.atomgit.com:443` 的网络连通性、代理配置及 SSL 中间件状态。

## 需要进一步确认的点
- 日志仅覆盖 trigger/编排层 job，未包含真正执行 Docker 构建的下游架构 job（`x86-64` / `aarch64`）日志；由于编排脚本因下载失败根本未运行，本次失败可确认发生在编排层。
- 需确认 `raw.atomgit.com:443` 的 SSL 失败是瞬时网络抖动还是持续性故障：建议查看该 pipeline 是否为首次出现、同批次其他 PR 是否同时失败。
- 若重跑后 trigger job 成功、下游 `x86-64` / `aarch64` 构建 job 才暴露真实错误，则需重新获取对应架构构建 job 的日志（`/job/x86-64/...` 或 `/job/aarch64/...`）再行分析。

## 修复验证要求
不涉及正则 patch 外部源文件。本次为 infra-error，无需 code-fixer 修改，建议直接重跑 CI 验证。

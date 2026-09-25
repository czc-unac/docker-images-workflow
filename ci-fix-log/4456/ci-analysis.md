# CI 失败分析报告

## 基本信息
- PR: #4456 — 【自动升级】pwdft容器镜像升级至52ad8cf版本.
- 失败类型: infra-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 构建脚本下载超时
- 新模式症状关键词: curl: (28), SSL connection timeout, build.sh, No such file or directory, Exit code 1

## 根因分析

### 直接错误
```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:00 --:--:--     0
...
  0     0    0     0    0     0      0      0 --:--:--  0:05:00 --:--:--     0
curl: (28) SSL connection timeout
chmod: cannot access 'build.sh': No such file or directory
/tmp/jenkins5688517101871300549.sh: line 21: ./build.sh: No such file or directory
Build step 'Execute shell' marked build as failure
Notifying upstream projects of job completion
Finished: FAILURE
```

### 根因定位
- 失败位置: 编排层 Jenkins 步骤 `/tmp/jenkins5688517101871300549.sh:21`（由 `multiarch/openeuler/trigger/openeuler-docker-images` build #4684 触发，运行在 `ecs-build-docker-x86-hk`）
- 失败原因: 该 job 首先用 `curl` 下载构建脚本（`build.sh`），下载过程持续约 5 分钟始终接收 0 字节，最终 `curl: (28) SSL connection timeout` 失败；由于脚本未下载成功，后续 `chmod build.sh` 与 `./build.sh` 均报 `No such file or directory`，job 被标记为 failure。真正的 Docker 镜像构建从未开始执行。

### 与 PR 变更的关联
无关。`pr.diff` 仅新增/修改了 `HPC/pwdft/52ad8cf/24.03-lts-sp4/Dockerfile`、`HPC/pwdft/README.md`、`HPC/pwdft/doc/image-info.yml`、`HPC/pwdft/meta.yml` 这些构建与元数据文件。日志显示失败发生在 CI 拉取构建脚本（`build.sh`）的网络阶段，尚未进入对上述文件的解析或 Docker 构建阶段。日志末尾明确为 `Finished: FAILURE`，不存在“日志成功但状态失败”的下游 job 误判情形，失败点即编排层网络下载。

## 修复方向

### 方向 1（置信度: 高）
判定为 CI 基础设施问题（`curl` 访问下载 `build.sh` 的源站发生 SSL 连接超时），与 PR 代码改动无关。Code Fixer 无需修改任何 Dockerfile 或元数据文件，由 CI 侧对该 job 重试/修复脚本下载通道即可。

### 方向 2（可选）
若重试后仍在同一 curl 步骤超时，说明下载源站或网络链路持续不可达，需由 CI 运维更换脚本分发源或排查 runner 出网/SSL 证书链路。

## 需要进一步确认的点
1. 该编排步骤中 `curl` 请求的完整 URL 与目标主机是什么（日志被截断，未打印 URL），以确认超时源站。
2. 该下载失败是单次网络抖动还是持续性问题——需查看同批次其他架构 job（如 aarch64）是否出现同样超时。
3. 确认 `build.sh` 的预期分发位置（制品仓库/CI 共享存储/内网服务），判断是临时不可达还是配置错误。

## 修复验证要求
不涉及正则 patch 外部源文件，无需填写。

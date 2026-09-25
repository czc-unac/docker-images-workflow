# CI 失败分析报告

## 基本信息
- PR: #4455 — 【自动升级】oceanbase容器镜像升级至5.0.1版本.
- 失败类型: infra-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 构建脚本拉取SSL失败
- 新模式症状关键词: curl (35), OpenSSL SSL_connect, SSL_ERROR_SYSCALL, raw.atomgit.com, build.sh, No such file or directory

## 根因分析

### 直接错误
```
curl: (35) OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to raw.atomgit.com:443
chmod: cannot access 'build.sh': No such file or directory
/tmp/jenkins2958517593238033126.sh: line 21: ./build.sh: No such file or directory
Build step 'Execute shell' marked build as failure
Notifying upstream projects of job completion
Finished: FAILURE
```

### 根因定位
- 失败位置: `/tmp/jenkins2958517593238033126.sh:21`（CI 编排脚本中从 `raw.atomgit.com` 下载 `build.sh` 的步骤）
- 失败原因: x86-64 构建 job 在真正执行 Docker 构建之前，先从 `raw.atomgit.com` 拉取通用构建脚本 `build.sh`，该 HTTPS 请求在持续约 62 秒后 TLS 握手失败（curl error 35 `SSL_ERROR_SYSCALL`，0 字节传输）。由于 `build.sh` 下载失败未落盘，后续 `chmod: cannot access 'build.sh'` 与 `./build.sh` 均报 “No such file or directory”，构建在 bootstrap 阶段即被标记失败。

### 与 PR 变更的关联
无关联。本 PR 仅新增了 `Database/oceanbase/5.0.1/24.03-lts-sp4/Dockerfile`，并在 `README.md`、`doc/image-info.yml`、`meta.yml` 中登记新版本条目。失败发生在拉取 CI 通用构建脚本的网络阶段，该脚本与 oceanbase 5.0.1 的 Dockerfile 内容无关；同一阶段的网络故障会影响任意 PR，属基础设施问题。

## 修复方向

### 方向 1（置信度: 高）
重跑 CI。该失败是构建脚本下载阶段的一次性网络/TLS 握手失败（`raw.atomgit.com:443` 请求 0 字节、随后 curl (35)），与 PR 代码无关。Code Fixer **无需修改任何文件**；由 CI 维护方重试流水线即可。

### 方向 2（置信度: 中）
若重试后仍在同一位置失败，则需由 CI 维护方排查 runner 到 `raw.atomgit.com` 的出网链路、DNS 解析及 CA 证书配置（`SSL_ERROR_SYSCALL` 常见于连接被中断或证书链校验前的 TCP 层异常），必要时将脚本下载源切换为可达镜像站。

## 需要进一步确认的点
- 该 SSL 失败是 `raw.atomgit.com` 侧的临时抖动，还是 CI runner 出网被限（需重试或网络抓包确认）。
- 是否是全局性问题（同批次其它 PR 是否也在拉取 `build.sh` 时失败）；若仅本 PR 失败，则应怀疑上游 trigger 层对该 PR 的调度差异。
- CI runner 的 CA 证书包是否完整。

## 修复验证要求
不适用。本失败为基础设施层网络错误，修复方向不涉及正则 patch 任何第三方/上游源文件，Code Fixer 无需执行外部源文件验证。

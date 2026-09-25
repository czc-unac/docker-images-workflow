# 修复摘要

## 修复的问题
CI 失败为基础设施层网络错误（拉取通用构建脚本 `build.sh` 时 TLS 握手失败），与 PR 代码无关，无需进行代码修改。

## 修改的文件
- 无

## 修复逻辑
分析报告将失败类型判定为 `infra-error`（置信度：高）。直接错误为：

```
curl: (35) OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to raw.atomgit.com:443
chmod: cannot access 'build.sh': No such file or directory
./build.sh: No such file or directory
```

失败发生在 CI 编排脚本从 `raw.atomgit.com` 下载通用构建脚本 `build.sh` 的网络阶段，传输 0 字节后 TLS 握手失败，导致脚本未落盘，后续 `chmod`/`./build.sh` 报 “No such file or directory”，构建在 bootstrap 阶段即被标记失败。

本 PR 的变更仅为新增 `Database/oceanbase/5.0.1/24.03-lts-sp4/Dockerfile` 并在 `README.md`、`doc/image-info.yml`、`meta.yml` 中登记新版本条目，与拉取 CI 通用脚本的网络步骤无关；同一网络故障会影响任意 PR，属基础设施问题。

因此按照分析报告方向 1，Code Fixer 无需修改任何文件，也未对 `pr.changed_files` 中的任何文件进行改动。建议由 CI 维护方重跑流水线；若重试后仍在同一位置失败，则由 CI 维护方排查 runner 到 `raw.atomgit.com` 的出网链路、DNS 与 CA 证书配置（方向 2）。

## 潜在风险
无。未修改任何源码文件，不影响 oceanbase 5.0.1 镜像构建逻辑。
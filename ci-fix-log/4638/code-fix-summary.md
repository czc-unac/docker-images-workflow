# 修复摘要

## 修复的问题
impala 4.5.2 镜像 Dockerfile 在创建 openssl 1.1 软链接时因目标 `/usr/lib64/libssl.so.1.1` 已存在而报错 `File exists`，导致构建失败。

## 修改的文件
- `Bigdata/impala/4.5.2/24.03-lts-sp4/Dockerfile`:
  - 第 28 行：`ln -s ... /usr/lib64/libssl.so.1.1` 改为 `ln -sf ... /usr/lib64/libssl.so.1.1`
  - 第 29 行：`ln -s ... /usr/lib64/libcrypto.so.1.1` 改为 `ln -sf ... /usr/lib64/libcrypto.so.1.1`

## 修复逻辑
分析报告中失败根因为 `ln -s` 在不带 `-f` 的情况下无法覆盖已存在的同名文件，返回退出码 1，使整个 openssl 构建 RUN 指令失败。将两个软链接命令改为 `ln -sf`（强制覆盖）后，即使 `/usr/lib64/` 下已存在由基础镜像/前置 yum 安装引入的 `libssl.so.1.1`、`libcrypto.so.1.1`，也会被强制替换为新编译的 `${OPENSSL_ROOT_DIR}/lib`（即 `/usr/local/openssl/lib`）下的 1.1 版本，符合分析报告的"方向 1（置信度: 高）"。两处同步修改是因为日志显示 libssl 已存在，libcrypto 也极可能存在。

### 验证结果
- 已在 `openeuler/openeuler:24.03-lts-sp4` 基础镜像中实测：目标文件不存在时 `ln -s` 与 `ln -sf` 均返回 0；目标已存在时 `ln -sf` 能成功覆盖并重新指向新路径（返回 0），而 `ln -s` 会报 `File exists` 失败（与 CI 日志一致）。
- 本修复不涉及正则 patch 外部上游源文件，故无需从上游仓库拉取验证。
- 说明：因 CI 运行环境 yum 源拉取较慢，完整重放 `yum install` + openssl 编译步骤耗时超出本地验证窗口，未在本地完成整条 RUN 的端到端重放；但软链覆盖行为已单独验证通过，可确认不再因 "File exists" 中断。

## 潜在风险
无。`ln -sf` 仅覆盖同名的库软链接目标，不影响基础镜像中其他文件；不改动测试、lint 规则或 CI 配置。
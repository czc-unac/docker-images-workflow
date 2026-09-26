# CI 失败分析报告

## 基本信息
- PR: #4556 — 【自动升级】impala容器镜像升级至4.5.2版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 符号链接已存在
- 新模式症状关键词: ln: failed to create symbolic link, File exists, libssl.so.1.1, exit code: 1

## 根因分析

### 直接错误
```
#10 50.52 ln: failed to create symbolic link '/usr/lib64/libssl.so.1.1': File exists
#10 ERROR: process "/bin/sh -c wget https://www.openssl.org/source/openssl-${OPENSSL_VERSION}.tar.gz -O /tmp/openssl-${OPENSSL_VERSION}.tar.gz &&     cd /tmp && tar -xvf openssl-${OPENSSL_VERSION}.tar.gz &&     cd openssl-${OPENSSL_VERSION} &&     ./config shared --openssldir=${OPENSSL_ROOT_DIR} --prefix=${OPENSSL_ROOT_DIR} &&     make -j$(nproc) && make install &&     ln -s ${OPENSSL_ROOT_DIR}/lib/libssl.so.1.1 /usr/lib64/libssl.so.1.1 &&     ln -s ${OPENSSL_ROOT_DIR}/lib/libcrypto.so.1.1 /usr/lib64/libcrypto.so.1.1 &&     rm -rf /tmp/*" did not complete successfully: exit code: 1
ERROR: failed to solve: process "..." did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Bigdata/impala/4.5.2/24.03-lts-sp4/Dockerfile:28`（RUN 步骤为第 23–30 行，OpenSSL 1.1.1g 构建阶段）
- 失败原因: `ln -s ${OPENSSL_ROOT_DIR}/lib/libssl.so.1.1 /usr/lib64/libssl.so.1.1` 未加 `-f` 强制覆盖；基础镜像 `openeuler/openeuler:24.03-lts-sp4` 中 `/usr/lib64/libssl.so.1.1` 已存在，导致 `ln` 报 `File exists` 并以 exit code 1 终止，`&&` 链中断，Docker 构建失败（其后的 `libcrypto.so.1.1` 链接步骤也未能执行）。

### 与 PR 变更的关联
本次 PR 新增了整个 `4.5.2/24.03-lts-sp4/Dockerfile`（含 OpenSSL 1.1.1g 的下载、编译与 `/usr/lib64` 符号链接步骤），失败步骤正是该新增文件中的第 23–30 行。失败与本次新增内容直接相关。基础镜像从旧版本（如 4.5.0 使用的 24.03-lts-sp2）升级到 24.03-lts-sp4 后，系统自带的 openssl 1.1.1 已在该路径提供目标文件，从而与 Dockerfile 中无条件创建同名符号链接的动作冲突。

> 说明：日志末尾为 `Finished: FAILURE`、`Build step 'Execute shell' marked build as failure`，非成功日志，故按真实失败分析。

## 修复方向

### 方向 1（置信度: 高）
在创建符号链接时允许覆盖已有文件：将 `ln -s` 改为 `ln -sf`，或在 `ln -s` 之前先 `rm -f` 目标路径。对 `libssl.so.1.1` 与 `libcrypto.so.1.1` 两处均需处理（当前链在第一条即中断，第二条尚未验证）。

### 方向 2（可选，置信度: 中）
评估是否仍需手动创建该符号链接。既然基础镜像已自带 `/usr/lib64/libssl.so.1.1`，若其版本/ABI 满足 impala 构建需求，可考虑删除这两条链接命令，避免与系统库冲突；若确实需要指向自编译的 1.1.1g，则应采用方向 1 的强制覆盖方式，并确认系统其他组件不依赖原链接。

## 需要进一步确认的点
- 基础镜像 `openeuler/openeuler:24.03-lts-sp4` 中 `/usr/lib64/libssl.so.1.1` 与 `/usr/lib64/libcrypto.so.1.1` 的实际来源与版本（是否为有效的 openssl 1.1.1 库、是否为悬空链接），以决定是覆盖还是跳过。
- 对照同仓库既有的 impala Dockerfile（如 `4.5.0/24.03-lts-sp2`）确认该 `ln -s` 步骤在旧基础镜像下是否曾成功，从而判断本次是基础镜像变化所致还是新增文件本身的缺陷。
- `libffi` 步骤中的 `ln -s /usr/local/lib64/libffi.so.6 /usr/lib64/libffi.so.6`（Dockerfile 约第 17 行）存在同类风险，建议一并检查是否需改为 `ln -sf`。

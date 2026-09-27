# CI 失败分析报告

## 基本信息
- PR: #4638 — 【自动升级】impala容器镜像升级至4.5.2版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 软链接目标已存在
- 新模式症状关键词: ln: failed to create symbolic link, File exists, libssl.so.1.1, Dockerfile exit code 1

## 根因分析

### 直接错误
```
#10 97.41 ln: failed to create symbolic link '/usr/lib64/libssl.so.1.1': File exists
#10 ERROR: process "/bin/sh -c wget https://www.openssl.org/source/openssl-${OPENSSL_VERSION}.tar.gz ... && make install && ln -s ${OPENSSL_ROOT_DIR}/lib/libssl.so.1.1 /usr/lib64/libssl.so.1.1 && ln -s ${OPENSSL_ROOT_DIR}/lib/libcrypto.so.1.1 /usr/lib64/libcrypto.so.1.1 && rm -rf /tmp/*" did not complete successfully: exit code: 1
Dockerfile:23
  22 |     ARG OPENSSL_ROOT_DIR=/usr/local/openssl
  23 | >>> RUN wget ... openssl-${OPENSSL_VERSION}.tar.gz ...
  28 | >>>     ln -s ${OPENSSL_ROOT_DIR}/lib/libssl.so.1.1 /usr/lib64/libssl.so.1.1 && \
  29 | >>>     ln -s ${OPENSSL_ROOT_DIR}/lib/libcrypto.so.1.1 /usr/lib64/libcrypto.so.1.1 && \
```

### 根因定位
- 失败位置: `Bigdata/impala/4.5.2/24.03-lts-sp4/Dockerfile:28`（openssl 构建 RUN 中的 `ln -s ... /usr/lib64/libssl.so.1.1`）
- 失败原因: 基础镜像 `openeuler/openeuler:24.03-lts-sp4` 中已存在 `/usr/lib64/libssl.so.1.1`，`ln -s`（无 `-f`，不覆盖已存在目标）创建符号链接失败并返回退出码 1，导致该 RUN 指令整体失败、Docker 构建中断。

### 与 PR 变更的关联
失败完全由本 PR 新增的 Dockerfile `Bigdata/impala/4.5.2/24.03-lts-sp4/Dockerfile` 触发。该文件为全新文件（`new_file: True`），其 openssl 构建步骤使用不带 `-f` 的 `ln -s` 直接向 `/usr/lib64/` 建立 `libssl.so.1.1` / `libcrypto.so.1.1` 软链，未处理基础镜像（或前置步骤）中已存在同名库文件的情况，因此在第一次执行到该软链步骤时即失败。PR 的其余改动（README.md、meta.yml、doc/image-info.yml、patches/build_impala.patch）均非本错误的直接原因。

## 修复方向

### 方向 1（置信度: 高）
在建立 `/usr/lib64/libssl.so.1.1` 与 `/usr/lib64/libcrypto.so.1.1` 符号链接前，先删除已存在的同名文件，或将 `ln -s` 改为强制覆盖形式（`ln -sf` / `ln -snf`），使软链指向新编译的 `/usr/local/openssl/lib` 版本。需同步确认该两步都采用覆盖策略，因为日志显示 libssl 已存在，libcrypto 大概率同样已存在。

### 方向 2（可选）
若基础镜像中已存在的 `libssl.so.1.1` 并非所需版本，更稳妥的做法是先 `rm -f` 目标路径再建立软链；若基础镜像本身已提供 openssl 1.1 运行时，也可评估是否可复用系统库而省略自编译软链步骤（需在验证时确认 impala 构建对此版本的实际依赖）。

## 需要进一步确认的点
- 需确认基础镜像 `openeuler/openeuler:24.03-lts-sp4` 中 `/usr/lib64/libssl.so.1.1`、`/usr/lib64/libcrypto.so.1.1` 的实际来源与版本（是否为系统 compat 包或前一构建层遗留），以决定"覆盖"还是"复用"策略。
- 需确认本 PR 之前已有版本的 impala Dockerfile（如 4.4.1/4.5.0）中该软链步骤的写法，判断是否为新 Dockerfile 相对旧模板引入的回归。
- 建议顺带核对 libffi 步骤中引用了未定义的 `${LIBFFI_ARGS}`（无对应 `ARG`）是否会影响后续步骤，但该点非当前失败根因。

## 修复验证要求
本修复方向（在 Dockerfile 中以 `rm -f` / `ln -sf` 处理已存在的 `/usr/lib64/libssl.so.1.1`、`/usr/lib64/libcrypto.so.1.1`）不涉及正则匹配外部上游源文件，故无需从上游拉取验证。code-fixer 必须实际触发一次 Docker 构建（至少在 openEuler 24.03-lts-sp4 基础镜像上执行到该 openssl RUN 步骤）确认软链步骤不再因 "File exists" 失败，并确认生成的两个软链正确指向 `/usr/local/openssl/lib` 下的 1.1 版本后再提交。

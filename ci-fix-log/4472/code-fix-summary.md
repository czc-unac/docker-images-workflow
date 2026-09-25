# 修复摘要

## 修复的问题
修复 Impala 4.5.2 镜像构建时，在 `/usr/lib64/` 创建 OpenSSL 符号链接因目标文件已存在（`File exists`）导致 `ln -s` 返回非 0、构建失败的问题。

## 修改的文件
- `Bigdata/impala/4.5.2/24.03-lts-sp4/Dockerfile`: 将 openssl 编译安装 RUN 步骤中创建 `libssl.so.1.1` 与 `libcrypto.so.1.1` 的两条 `ln -s` 改为 `ln -sf`，使链接创建具备幂等性（目标已存在时强制覆盖）。

## 修复逻辑
分析报告根因：openssl 1.1.1g 编译安装成功后的收尾步骤执行
`ln -s ${OPENSSL_ROOT_DIR}/lib/libssl.so.1.1 /usr/lib64/libssl.so.1.1`，
而基础镜像 `openeuler/openeuler:24.03-lts-sp4` 的 `/usr/lib64/` 下已存在同名文件/链接，`ln -s` 默认不覆盖已存在文件，直接返回非 0，使整个 RUN 步骤以 exit code 1 失败。

采用分析报告"方向 1（置信度 高）"：让两条链接的创建具备幂等性。使用 `ln -sf` 强制覆盖目标路径下的同名链接/文件，既保证链接指向新编译的 `/usr/local/openssl/lib`（满足 Impala 对 OpenSSL 1.1.1 的依赖），又避免"File exists"失败，且该 RUN 步骤可重复执行。同一 RUN 中两条 `ln` 已一并处理。此改动不影响 libffi 步骤（该步骤在失败构建中已成功，位于 openssl 步骤之前）。本次修复不涉及对第三方源文件的正则 patch，无需从上游验证正则。

## 潜在风险
无。`ln -sf` 仅在此处覆盖 `/usr/lib64/` 下的 `libssl.so.1.1`、`libcrypto.so.1.1` 两个链接，目标均为本次新编译安装的 OpenSSL 1.1.1g，版本与依赖一致；改动不涉及其他文件或构建步骤。
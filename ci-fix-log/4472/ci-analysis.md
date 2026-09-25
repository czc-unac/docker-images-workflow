# CI 失败分析报告

## 基本信息
- PR: #4472 — 【自动升级】impala容器镜像升级至4.5.2版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 符号链接已存在
- 新模式症状关键词: `ln: failed to create symbolic link`, `File exists`, `libssl.so.1.1`, `exit code: 1`

## 根因分析

### 直接错误
```
#10 100.6 ln: failed to create symbolic link '/usr/lib64/libssl.so.1.1': File exists
#10 ERROR: process "/bin/sh -c wget https://www.openssl.org/source/openssl-${OPENSSL_VERSION}.tar.gz ... &&
    make -j$(nproc) && make install &&
    ln -s ${OPENSSL_ROOT_DIR}/lib/libssl.so.1.1 /usr/lib64/libssl.so.1.1 &&
    ln -s ${OPENSSL_ROOT_DIR}/lib/libcrypto.so.1.1 /usr/lib64/libcrypto.so.1.1 &&
    rm -rf /tmp/*" did not complete successfully: exit code: 1
ERROR: failed to solve: process "... ln -s ${OPENSSL_ROOT_DIR}/lib/libssl.so.1.1 /usr/lib64/libssl.so.1.1 ..." did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Bigdata/impala/4.5.2/24.03-lts-sp4/Dockerfile:28`（openssl 编译安装 RUN 步骤中的 `ln -s` 行）
- 失败原因: 编译安装 openssl 1.1.1g 后的收尾步骤用 `ln -s` 在 `/usr/lib64/` 下创建 `libssl.so.1.1` 符号链接，但目标路径下该文件/链接**已经存在**（基础镜像 `openeuler/openeuler:24.03-lts-sp4` 自带，或此前层已创建）。`ln -s` 默认不覆盖已存在文件，直接返回非 0，导致整个 RUN 步骤失败。openssl 本身的 wget / tar / config / make / make install 均已成功（日志显示 man page 安装到 100.6s），失败发生在最后创建链接阶段。

### 与 PR 变更的关联
本次 PR 新增了 `Bigdata/impala/4.5.2/24.03-lts-sp4/Dockerfile`（全新文件），该 openssl 构建 RUN 步骤即由此文件引入，因此失败**由本 PR 直接触发**。与该新 Dockerfile 一同提交的 `patches/build_impala.patch`、README、image-info.yml、meta.yml 改动与本次失败无直接关系。

## 修复方向

### 方向 1（置信度: 高）
在创建符号链接前先清理目标路径下可能已存在的同名文件，或改用可强制覆盖的链接方式，避免"File exists"导致 `ln` 返回失败。即让 `libssl.so.1.1` / `libcrypto.so.1.1` 两个链接的创建具备幂等性（先删除再创建，或强制覆盖）。同时建议对同一 RUN 中的两条 `ln` 一视同仁处理，防止修复第一条后第二条仍报同样错误。

### 方向 2（可选）
若确认为基础镜像已自带兼容的 `libssl.so.1.1` / `libcrypto.so.1.1`，也可直接跳过该链接创建步骤（仅当确认基础镜像中的版本满足 Impala 运行需求时）。

## 需要进一步确认的点
- 确认 `openeuler/openeuler:24.03-lts-sp4` 基础镜像 `/usr/lib64/` 下是否已存在 `libssl.so.1.1`、`libcrypto.so.1.1`（以及是符号链接还是真实文件），以决定采用"删除后重建"还是"跳过创建"。
- 确认基础镜像自带的 `libssl.so.1.1` 版本是否能满足 Impala 运行/编译对 OpenSSL 1.1.1 的依赖；若版本不符，仍应确保指向新编译的 `/usr/local/openssl/lib`。

## 修复验证要求
无需从上游仓库验证正则匹配（本次修复不涉及对第三方源文件的 patch）。code-fixer 若采用"删除后重建"方案，需保证在同一 Dockerfile 层内对 `libssl.so.1.1` 与 `libcrypto.so.1.1` 两条链接均做幂等处理，并确认修复后该 RUN 步骤可重复执行。

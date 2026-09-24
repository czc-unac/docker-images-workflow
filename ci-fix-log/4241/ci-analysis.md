# CI 失败分析报告

## 基本信息
- PR: #4241 — 【问题修复】milvus容器镜像升级至3.0.2版本
- 失败类型: dependency-error（OpenSSL / libldap 符号版本冲突引发的构建失败）
- 置信度: 中
- 知识库匹配: 新模式
- 新模式标题: OpenSSL符号版本冲突
- 新模式症状关键词: symbol lookup error, EVP_md2, libldap.so.2, OPENSSL_3.0.0, cmake, conan

## 根因分析

### 直接错误
日志中最早、最关键的错误是 `cmake` 启动时因符号查找失败而崩溃：

```
#13 3034.8 aws-c-cal/0.9.14: Running CMake.configure()
#13 3034.8 aws-c-cal/0.9.14: RUN: cmake -G "Unix Makefiles" -DCMAKE_TOOLCHAIN_FILE="generators/conan_toolchain.cmake" ...
#13 3034.8 cmake: symbol lookup error: /lib64/libldap.so.2: undefined symbol: EVP_md2, version OPENSSL_3.0.0
#13 3034.8 aws-c-cal/0.9.14: ERROR:
#13 3034.8 Package '9f90ae9f102bf5428e704efcea68df1eb2d6c4c8' build failed
#13 3034.8 ERROR: aws-c-cal/0.9.14: Error in build() method, line 78
#13 3034.8 	cmake.configure()
#13 3034.8 	ConanException: Error 127 while executing
#13 3034.8 make: *** [Makefile:305: build-3rdparty] Error 1
```

上游报错链：`make build-3rdparty`（Makefile:305）→ conan 构建 `aws-c-cal/0.9.14` → `cmake.configure()`（conanfile 第 78 行）→ 系统 `cmake` 无法启动。`Error 127` 只是 cmake 崩溃后的连带结果，真正根因是那行 `symbol lookup error`。

### 根因定位
- 失败位置: `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile:22-30`（新增的 `RUN git clone … make build-cpp && make build-go` 步骤），实际失败点在下游 conan 构建 `aws-c-cal/0.9.14` 的 `CMake.configure()` 阶段。
- 失败原因: 动态链接符号版本冲突。系统 `/lib64/libldap.so.2` 依赖 `EVP_md2@OPENSSL_3.0.0`（属于系统 OpenSSL 3.0），但在构建 `aws-c-cal` 时 conan 注入了它自己的 openssl 依赖（日志：`aws-c-cal/0.9.14: requires: … openssl/3.3.Z`，以及 `[HOOK - hook_fix_shared_lib_env.py] pre_build(): Set LD_LIBRARY_PATH for 3 dependency lib dirs`），使系统 `cmake` 进程实际加载到 conan 版 `libcrypto.so.3`。该 libcrypto 未提供 `EVP_md2`（或符号版本不一致），导致 libldap 解析失败，cmake 在真正执行前即崩溃。

关键旁证（支持"按包注入 LD_LIBRARY_PATH 才是触发点"这一判断）：
- 同一日志中，更早构建的 `rocksdb/6.29.5@milvus/dev` 使用 `cmake` 完成了构建与 `cmake --install`，均成功；`libxml2`、`automake` 等包也构建成功。
- 只有需要 `openssl/3.3.Z` 的 `aws-c-cal` 在 `pre_build` 阶段被 conan hook 设置了依赖库路径，随后其 `cmake` 立刻因 libldap→libcrypto 冲突崩溃。
- 这说明系统 OpenSSL 本身对无 openssl 依赖的包是可用的，问题出在 conan 为特定包"覆盖"了 OpenSSL 运行库。

### 与 PR 变更的关联
- 本 PR 新增了全新的 `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`（57 行新增）以及 `meta.yml` / `README.md` / `doc/image-info.yml` 条目。失败完全发生在这个新 Dockerfile 的 build 步骤内，因此与本次 PR 直接相关（构建新镜像即会失败）。
- 但触发失败的**不是** diff 中的三处 `sed` 适配（`ID=rocky`、去掉 epel/dnf-plugins-core、去掉 lcov），而是 milvus 3.0.2 的 conan 依赖链（`aws-c-cal` → `openssl/3.3.Z`）与 openEuler 24.03-lts-sp4 基础镜像系统 `openldap`/OpenSSL 3.0 之间的运行库冲突。
- 由于日志显示这是第一次为 3.0.2 版本建模（`new_file: True`），无法从日志判断旧版本(2.5.14/2.6.0)是否遇到同样问题；需在代码库中对比旧 Dockerfile 是否绕过了该依赖。

## 修复方向

### 方向 1（置信度: 中）
消除 conan OpenSSL 对系统 `cmake` 运行库的污染：让构建 `aws-c-cal` 时的 `cmake` 进程使用系统 OpenSSL，而不是 conan 的 `openssl/3.3`。可行思路包括调整/移除 conan `hook_fix_shared_lib_env.py` 对 `LD_LIBRARY_PATH` 的注入（或仅对 openssl 目录不注入）、在 Dockerfile 中通过环境变量调整库搜索顺序，或让 conan 直接使用系统 OpenSSL。核心目标是避免出现两份 `libcrypto.so.3`。

### 方向 2（置信度: 中）
让系统 `openldap`/`libldap` 与 OpenSSL 3.3 兼容，从而不再引用 `EVP_md2@OPENSSL_3.0.0`。即在构建阶段升级/对齐系统 openldap 及 openssl 相关包（例如更新 openldap 或补齐匹配的 `-devel` 包），使 `libldap.so.2` 不再依赖系统 OpenSSL 3.0 的旧符号。

### 方向 3（置信度: 低）
约束 conan 依赖的 OpenSSL 版本与基础镜像系统版本一致，避免双 libcrypto 并存；但需确认 milvus 3.0.2 的 conan recipe 是否允许锁定 openssl 版本且不影响其它 3rdparty 包。

> 说明：`dnf` 层面的系统 OpenSSL 升级/安装属于修复手段，但方向 1 与方向 2 存在耦合，需先做证据确认再决定。

## 需要进一步确认的点
日志不足以完全确定是"conan 注入 LD_LIBRARY_PATH"还是"系统 OpenSSL 被 install_deps.sh 升级"哪一种为主导，需在代码库/基础镜像中确认：
1. 基础镜像 `openeuler/openeuler:24.03-lts-sp4` 中 `/lib64/libldap.so.2` 与 `libcrypto.so.3` 的版本，以及 `libldap` 对 `EVP_md2@OPENSSL_3.0.0` 的具体引用（`objdump -T /lib64/libldap.so.2 | grep EVP_md2`、`rpm -q openldap openssl`）。
2. conan `openssl/3.3.Z` 包是否导出 `EVP_md2`、其符号版本与共享库路径。
3. milvus 源码中 `hook_fix_shared_lib_env.py`（`pre_build()` 设置 LD_LIBRARY_PATH 的具体内容）与 `aws-c-cal` conanfile 第 78 行 `cmake.configure()` 的上下文。
4. `scripts/install_deps.sh` 在 `ID=rocky` 适配后是否会安装/升级 openssl 等包（判断方向 2 是否成立）。
5. 仓库中既有的 `Database/milvus/2.5.14`、`2.6.0` Dockerfile 是否包含针对该冲突的额外处理，可作对照。

## 修复验证要求
（本次修复方向不涉及"正则 patch 外部源文件"，但置信度为"中"，code-fixer 在提交前必须完成以下验证）
1. 在与基础镜像一致的环境（openEuler 24.03-lts-sp4）中复现：确认 `cmake` 报 `libldap.so.2: undefined symbol: EVP_md2`，且该现象仅在构建带 `openssl/3.3.Z` 依赖的 `aws-c-cal` 时出现。
2. 采用修复后，确认 `make build-3rdparty` 能通过 `aws-c-cal/0.9.14` 阶段（不再出现 `cmake: symbol lookup error` 与 `Error 127`）。
3. 确认修复不会破坏此前已成功的 `rocksdb`、`libxml2`、`automake` 等 3rdparty 包，也不会引入新的 `ldd`/符号缺失问题。
4. 若选择方向 2（升级系统 openldap/openssl），需确认镜像内业务组件（LDAP 相关）兼容，并通过 amd64、arm64 两个架构的构建验证。

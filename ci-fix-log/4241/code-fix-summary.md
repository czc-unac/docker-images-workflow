# 修复摘要

## 修复的问题
修复 milvus 3.0.2 镜像构建时，conan 的 `openssl/3.3.2` 将自身 `libcrypto.so.3` 注入 `LD_LIBRARY_PATH`，污染系统 `cmake` 的运行库，导致 `libldap.so.2: undefined symbol: EVP_md2, version OPENSSL_3.0.0` 符号冲突、`make build-3rdparty` 在 `aws-c-cal/0.9.14` 阶段崩溃的问题。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`: 在 `git clone` 后的 `sed` 适配链中新增一条 `sed`，修改 `scripts/3rdparty_build.sh` 生成的 conan `pre_build` hook，使 `openssl` 依赖的 lib 目录不再被注入 `LD_LIBRARY_PATH`（其余依赖不受影响）。

## 修复逻辑
CI 报错链为：`make build-3rdparty` → conan 构建 `aws-c-cal/0.9.14` → `cmake.configure()` → `cmake: symbol lookup error: /lib64/libldap.so.2: undefined symbol: EVP_md2, version OPENSSL_3.0.0`。

根因：milvus 3.0.2 的 `scripts/3rdparty_build.sh` 会在 conan home 动态生成 `hook_fix_shared_lib_env.py`，其 `pre_build()` 遍历当前包的所有依赖，把每个依赖的 `lib` 目录**前置**到 `LD_LIBRARY_PATH`，用于让 `grpc_cpp_plugin` 等构建期工具找到共享库。构建 `aws-c-cal` 时该包依赖 conan `openssl/3.3.2`，于是 conan 版 `libcrypto.so.3` 被置于搜索路径最前。系统 `cmake`（openEuler 的 cmake 通过 `libcurl` 依赖 `/lib64/libldap.so.2`）启动时，`libldap` 需要 `EVP_md2@OPENSSL_3.0.0`，但被解析到 conan 的 `libcrypto.so.3`（该库不导出此符号），故动态链接失败、cmake 立即崩溃（`Error 127` 只是连带结果）。

修复方式与 CI 分析报告的“方向 1（仅对 openssl 目录不注入）”一致：让 hook 在收集 `dep_lib_dirs` 时跳过 `openssl` 依赖。这样 `cmake` 会回退使用系统 OpenSSL，`libldap` 的符号可正常解析；而 hook 原本要服务的构建期工具（protoc/grpc 插件等）与 openssl 无关，不受影响。

**正则 patch 外部源文件验证**：已从上游 `milvus-io/milvus` tag `v3.0.2` 获取 `scripts/3rdparty_build.sh`，在内存中实际执行该 `sed` 并验证：
- 目标字符串 `dep_lib_dirs.append(lib_dir)` 在文件中出现 2 处（分别位于 `hook_fix_shared_lib_env.py` 与 `hook_fix_macos_rpaths.py`），`sed` 后均被替换为带 `openssl` 过滤的版本；
- 提取替换后的两个 heredoc Python 代码块，`compile()` 校验语法均通过；
- 用模拟的 `conanfile.dependencies` 运行 `pre_build()`，结果 `LD_LIBRARY_PATH` 中排除了 `openssl`、保留了 `zlib/protobuf/aws-c-common`，过滤逻辑符合预期。

> 说明：上游 conan2 缓存目录名仅取包名前 5 个字符（`ref.name[:5]`），因此不能通过路径字符串判断是否 openssl；本修复改为通过依赖的 `dep.ref.name` 精确判断，避免误判。

## 潜在风险
- 该 `sed` 依赖上游 `scripts/3rdparty_build.sh` 中 `dep_lib_dirs.append(lib_dir)` 字面量；Dockerfile 固定 `ARG VERSION=3.0.2`，已按该 tag 验证匹配成功，故风险可控。若上游在该 tag 下无法访问或未来版本改动此行，`sed` 不匹配时会静默不生效（sed 退出码仍为 0），构建会回退到原行为。
- 排除 openssl 后，若有 conan 构建期工具显式依赖 conan 版 `libcrypto` 运行，将改为加载系统 OpenSSL（3.x ABI 兼容）；此类工具在该依赖链中未出现，实际影响很小。
- 未改动 `README.md`、`doc/image-info.yml`、`meta.yml`，其他文件与本次 CI 失败无关。
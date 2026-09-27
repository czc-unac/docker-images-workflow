# CI 失败分析报告

## 基本信息
- PR: #4658 — 【自动升级】ceph容器镜像升级至21.3.0版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式10
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#12 242.3 CMake Error at /usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:233 (message):
#12 242.3   Could NOT find Protobuf (missing: Protobuf_LIBRARIES Protobuf_INCLUDE_DIR)
#12 242.3 Call Stack (most recent call first):
#12 242.3   /usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:603 (_FPHSA_FAILURE_MESSAGE)
#12 242.3   /usr/share/cmake/Modules/FindProtobuf.cmake:772 (FIND_PACKAGE_HANDLE_STANDARD_ARGS)
#12 242.3   src/CMakeLists.txt:1029 (find_package)
#12 242.3 -- Configuring incomplete, errors occurred!
#12 242.3 + exit 1
#12 ERROR: process "/bin/sh -c git clone -b v${VERSION} --recursive --depth 1 https://github.com/ceph/ceph.git ... && ninja install" did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile:40`（ceph 构建 RUN 步骤），CMake 报错点位于 ceph 源码 `src/CMakeLists.txt:1029` 的 `find_package(Protobuf ...)`
- 失败原因: Dockerfile 第一个 `dnf install` 步骤的依赖列表中**遗漏了 Protobuf 开发包**。CMake 配置阶段 `find_package(Protobuf)` 找不到 `Protobuf_LIBRARIES` / `Protobuf_INCLUDE_DIR`，配置中止，`do_cmake.sh` 返回 exit 1，整个 ceph 构建层失败。

### 与 PR 变更的关联
直接相关。本 PR 新增 `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`，将 ceph 升级到 21.3.0。该版本的 ceph 构建系统（`src/CMakeLists.txt:1029`）强依赖 Protobuf，而 Dockerfile 的 `dnf install` 清单中未包含对应 `-devel` 包（清单中可见 `thrift-devel`、`openldap-devel`、`cryptsetup-devel` 等，但无 protobuf 相关项），因此新版本构建必然在此处失败。旧版本 20.3.0 Dockerfile 未触发此问题。

### 次要信息（非根因，不影响本次判定）
- 日志中 `#12 242.0 bash: line 1: jq: command not found` 为非致命提示，构建仍继续。
- `UndefinedVar: Usage of undefined variable '$LD_LIBRARY_PATH' (line 47)` 与 `FromAsCasing` 仅为 BuildKit 警告（对应模式20），非本次失败原因。
- 日志末尾为 `Finished: FAILURE`，与 `ci_failed` 状态一致，失败真实发生于本次提供的 ceph 构建 job 中。

## 修复方向

### 方向 1（置信度: 高）
在 `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile` 的第一个 `dnf install -y` 依赖清单中补充 Protobuf 相关开发包，使 CMake 的 `find_package(Protobuf)` 能定位到库与头文件。至少需包含提供 `Protobuf_LIBRARIES` 与 `Protobuf_INCLUDE_DIR` 的 `protobuf-devel`，以及提供 `protoc` 的 `protobuf-compiler`（ceph 构建需要 protoc 生成代码）。包名需以 openEuler 24.03-LTS-SP4 仓库实际提供的名称为准。

### 方向 2（可选）
除补包外，可顺带消除日志中的 BuildKit 警告：将 `ENV LD_LIBRARY_PATH=/usr/local/lib64:$LD_LIBRARY_PATH` 改为使用默认值语法（`${LD_LIBRARY_PATH:-}`），以避免 `UndefinedVar` 警告。此为可选清理项，不是本次失败根因。

## 需要进一步确认的点
1. openEuler 24.03-LTS-SP4 仓库中 Protobuf 的实际包名组合（如 `protobuf-devel` + `protobuf-compiler`，或还需 `protobuf`/`protobuf-c`）。修复前应确认包名，避免再次因包不存在而失败。
2. ceph 21.3.0 是否还依赖其他本次未安装的库（本次日志在 Protobuf 处即中止，后续依赖是否齐全无法从现有日志判断）。建议补包后重新构建，观察 cmake 是否报出下一个缺失依赖。
3. 是否需同步核对 `20.3.0` 与 `21.3.0` 两个 Dockerfile 的依赖差异，确认升级带来的新增构建依赖已全部补齐。

## 修复验证要求
本修复方向为补装系统依赖包，不涉及对第三方/上游源文件的正则 patch，故无需额外的上游源码正则匹配验证。但 code-fixer 在提交前，应从 openEuler 24.03-LTS-SP4 的仓库元数据确认 `protobuf-devel` / `protobuf-compiler` 包真实存在，避免包名错误导致新的 dependency-error。

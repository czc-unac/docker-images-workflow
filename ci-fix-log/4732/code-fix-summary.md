# 修复摘要

## 修复的问题
ceph 21.3.0 镜像构建时，源码 `ninja` 全量编译在 dashboard 前端 nodeenv 步骤失败（`chown: cannot access '.../node-env/src': No such file or directory`），通过在对 ceph 源码执行 `do_cmake.sh` 时增加 `-DWITH_MGR_DASHBOARD_FRONTEND=OFF` 跳过该前端构建。

## 修改的文件
- `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`: 在 `do_cmake.sh` 的 cmake 参数中追加 `-DWITH_MGR_DASHBOARD_FRONTEND=OFF`。

## 修复逻辑
本次并非依据上下文提供的“证据不足”分析报告（该报告对应原始 PR 的首个失败，缺少日志），而是直接获取了真实 CI 日志后定位根因：

1. **原始 PR 首个失败（Protobuf）已被既有提交修复**：从 openEuler 门禁评论取得原始 PR #4732 的构建日志（x86-64 #4811 / aarch64 #4907），最早错误为
   `CMake Error ... Could NOT find Protobuf (missing: Protobuf_LIBRARIES Protobuf_INCLUDE_DIR)`（`src/CMakeLists.txt:1029`）。
   该 `find_package(Protobuf REQUIRED)` 位于 `if(WITH_NVMEOF_GATEWAY_MONITOR_CLIENT)` 块内（上游 v21.3.0 `src/CMakeLists.txt:1021-1029`，该选项默认 ON），因此现分支上已有的 `-DWITH_NVMEOF_GATEWAY_MONITOR_CLIENT=OFF`（前一轮修复）已使该错误消失。

2. **当前 fix 分支失败根因**：从修复 PR #4745 的门禁评论取得构建日志（x86-64 #4833 / aarch64 #4929），cmake 配置阶段已通过，失败发生在 `ninja` 编译阶段，唯一失败目标为：
   ```
   FAILED: [code=1] src/pybind/mgr/dashboard/frontend/node-env/bin/npm ...
   chown: cannot access '/opt/ceph/build/src/pybind/mgr/dashboard/frontend/node-env/src': No such file or directory
   ninja: build stopped: subcommand failed.
   ```
   该步骤由 ceph 上游 `src/pybind/mgr/dashboard/frontend/CMakeLists.txt:68-80` 的 nodeenv 自定义命令触发：安装 nodeenv 1.11.0 后 `node-env/src` 目录已不存在，导致 `chown` 返回非 0。两架构报错完全一致。

3. **修复方式**：ceph 上游 `src/pybind/mgr/dashboard/CMakeLists.txt` 中前端构建由 `if(WITH_MGR_DASHBOARD_FRONTEND)` 保护，且该选项在 v21.3.0 根 `CMakeLists.txt:831` 定义为 `option(... ON)`。传入 `-DWITH_MGR_DASHBOARD_FRONTEND=OFF` 即可完全不进入前端 nodeenv 构建。该容器仅运行 `entrypoint.sh` 启动单节点 mon，不使用 dashboard/mgr，关闭前端无功能影响，且比修改上游源码或锁定 nodeenv 版本更稳定、改动最小。

> 验证：已从 Jenkins 实际构建日志确认上述失败（x86-64 #4833、aarch64 #4929），并从上游 `raw.githubusercontent.com/ceph/ceph/v21.3.0/` 获取 `CMakeLists.txt`、`src/CMakeLists.txt`、`src/pybind/mgr/dashboard/CMakeLists.txt`、`src/pybind/mgr/dashboard/frontend/CMakeLists.txt` 核实选项名与行为。

## 潜在风险
- 关闭 dashboard 前端后，镜像不包含 ceph dashboard 的 Web UI 静态资源。本镜像的用途是验证上游 ceph 版本与 openEuler 的集成（`entrypoint.sh` 只启动 mon 并用 `ceph -s` 校验），不使用 dashboard，无影响。
- 该改动仅针对 ceph 21.3.0 的 Dockerfile，不影响 20.3.0 等既有版本；未改动任何其他文件。
- 全量编译规模较大（1763 个目标），若后续仍存在其他独立的编译问题，需要新的构建日志才能定位，本次仅修复已证实的 nodeenv 失败。
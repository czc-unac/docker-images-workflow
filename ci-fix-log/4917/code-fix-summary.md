# 修复摘要

## 修复的问题
ceph 21.3.0 镜像构建在 `do_cmake.sh` 配置阶段失败后，继续在编译阶段失败：ceph v21.3.0 的
dashboard 前端构建会无条件执行 `chown -R ... node-env/src`，而构建时通过未固定版本的
`pip install nodeenv` 装到 nodeenv 1.11.0（2026-09-26 发布），该版本默认会用 `clean_src=True`
删除 `node-env/src`，导致 `chown: cannot access '.../node-env/src': No such file or directory`，
`ninja` 报 `build stopped: subcommand failed`，x86_64 与 aarch64 均失败。

## 修改的文件
- `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`: 在 `git clone` 之后、`./do_cmake.sh` 之前新增一行
  `sed`，把上游 `src/pybind/mgr/dashboard/frontend/CMakeLists.txt` 中无条件的 `chown` 改为
  “`src` 存在时才 chown”（与上游官方修复一致），使 dashboard 前端可正常构建。
- （当前分支已有的前置修复，保留）：`dnf install` 列表中的
  `protobuf-devel protobuf-compiler grpc-devel grpc-plugins`，用于修复上一层面的
  `CMake Error ... Could NOT find Protobuf`。

## 修复逻辑
分析报告结论为“证据不足/无法定位根因”，其最可疑方向（上游不存在 tag `v21.3.0`）经核实**不成立**：
`git ls-remote` 与 GitHub API 均确认 `ceph/ceph` 存在 tag `v21.3.0`
（指向 main 上 commit `b44498fd`，2026-06-10）。因此本次修复改用真实 CI 日志定位根因：

1. 从 PR #4917 评论中提取到真实失败 job，并取得失败构建日志：
   - 原始失败（无 protobuf 修复）：x86_64 job 5036、aarch64 job 5132
     → 报错 `src/CMakeLists.txt:1029 find_package` → `Could NOT find Protobuf`。
     该问题对应 `WITH_NVMEOF_GATEWAY_MONITOR_CLIENT=ON`（默认开）下的
     `find_package(Protobuf REQUIRED)`，已由分支上已有的 `protobuf/grpc` 依赖修复。
   - 前置修复后的失败（fix PR #4926，head `4b9a7a5b0`）：x86_64 job 5045、aarch64 job 5141
     → 日志中**唯一**的 `FAILED:` 边为
     `src/pybind/mgr/dashboard/frontend/node-env/bin/npm`，原因是
     `chown: cannot access '.../node-env/src'`。cmake 配置阶段已通过，证明 protobuf/grpc 修复有效，
     剩余失败为 nodeenv 行为变更。
2. 上游定位：nodeenv 1.11.0 的 `Config.clean_src` 默认 `True`，安装后会删除 `src`；而
   ceph v21.3.0 的 `frontend/CMakeLists.txt` 无条件 `chown .../src`。上游已用 commit
   `5aff221610`（“mgr/dashboard: tolerate nodeenv leaving no src directory behind”）修复，
   将命令改为 `test ! -e ${mgr-dashboard-nodeenv-dir}/src || chown -R ...`。
3. 由于镜像固定使用 tag `v21.3.0`（该 tag 早于上游修复），在 Dockerfile 中对克隆出的源码施加与
   上游完全一致的一行补丁。

**正则 patch 验证结果**：已从上游 `v21.3.0` 获取
`src/pybind/mgr/dashboard/frontend/CMakeLists.txt`（`https://raw.githubusercontent.com/ceph/ceph/v21.3.0/src/pybind/mgr/dashboard/frontend/CMakeLists.txt`），
并在内存中用 Dockerfile 中完全相同的 `sed` 命令测试：
`re`/`sed` 匹配成功，替换后的行与上游修复 commit `5aff221610` 的结果**逐字一致**。
（同时确认根 `CMakeLists.txt:831` 选项 `WITH_MGR_DASHBOARD_FRONTEND` 默认 ON，故该目标在 `ninja` 默认构建中，必须处理。）

说明：20.3.0 的同一 dashboard 路径在本仓 CI 中可正常构建，且其 Dockerfile 同样未安装 `jq`
（日志中的 `jq: command not found` 为非致命告警），故本次不额外引入 `jq`，保持最小改动。

## 潜在风险
- 该修复依赖上游 `frontend/CMakeLists.txt` 的确切行文本，已针对 tag `v21.3.0` 验证匹配；
  若后续升级到上游已修复的版本，此 `sed` 将不再匹配但对构建无害（`sed` 无匹配时返回 0，不会失败）。
- dashboard 前端（npm/Angular）构建量较大且依赖 npm 源；如该目标在 CI 中出现其它问题，
  可能需要在后续迭代进一步处理。核心 ceph 组件与 `entrypoint.sh`（仅启动 ceph-mon）不受影响。
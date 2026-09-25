# CI 失败分析报告

## 基本信息
- PR: #4486 — 【自动升级】onnxruntime容器镜像升级至1.30.0版本.
- 失败类型: dependency-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: gcc工具链包名错误
- 新模式症状关键词: Unable to find a match, No match for argument, gcc-toolset-14-c++, yum install, 通配符包名

## 根因分析

### 直接错误
```
#7 46.62 No match for argument: gcc-toolset-14-c++*
#7 46.65 Error: Unable to find a match: gcc-toolset-14-c++*
#7 ERROR: process "/bin/sh -c yum update -y &&     yum install -y         tar         ca-certificates         gcc-toolset-14-gcc*         gcc-toolset-14-binutils*         gcc-toolset-14-c++*         python3-devel ..." did not complete successfully: exit code: 1
ERROR: failed to solve: process "/bin/sh -c yum update -y && ..." did not complete successfully: exit code: 1
Dockerfile:8
```

### 根因定位
- 失败位置: `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile`:14
- 失败原因: `yum install` 列表中 `gcc-toolset-14-c++*` 这一项在 openEuler 24.03-LTS-SP4 仓库中不存在对应 RPM 包，yum 无法匹配该通配符参数，直接以 `Unable to find a match` 报错并终止安装（exit code 1）。

### 与 PR 变更的关联
本 PR 新增了 `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile`（全新文件），失败步骤即该文件 builder 阶段的第 2 个 RUN（yum 安装构建依赖）。因此该失败为本 PR 新增文件直接触发，与其它既有镜像无关。

补充说明：
- 日志中 yum 已完成 `yum update -y`（升级 38 个包，输出 `Complete!`），说明网络与仓库可用，失败点纯粹是包名不存在。
- 日志为 Docker build 的完整输出，末尾为 `Build step 'Execute shell' marked build as failure` / `Finished: FAILURE`，**不是** trigger 层假成功日志，可排除"日志与状态不一致"情形。
- `FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 4/40)` 仅为 warning，不致命，与本失败无关。

## 修复方向

### 方向 1（置信度: 高）
修正 gcc-toolset-14 的 C++ 依赖包名。openEuler/RHEL 系中 gcc-toolset 的 C++ 编译器实际包名为 `gcc-toolset-14-gcc-c++`（会被 `gcc-toolset-14-gcc*` 通配符覆盖），而非 `gcc-toolset-14-c++`。因此应：
- 将 `gcc-toolset-14-c++*` 改为正确的 `gcc-toolset-14-gcc-c++`（或 `gcc-toolset-14-gcc-c++*`）；或
- 由于 `gcc-toolset-14-gcc*` 已能匹配到 `gcc-toolset-14-gcc-c++`，可考虑直接删除 `gcc-toolset-14-c++*` 这一冗余且错误的条目。

### 方向 2（可选）
核对本 PR 目标 openEuler 24.03-LTS-SP4 仓库中 gcc-toolset-14 系列实际提供的包清单，确认 `gcc-toolset-14-gcc*`、`gcc-toolset-14-binutils*` 通配符均能匹配到至少一个包，避免修正后再次因某个通配符无匹配而失败。

## 需要进一步确认的点
- 确认 openEuler 24.03-LTS-SP4 的 `EPOL`/`everything` 仓库中 gcc-toolset-14 系列的确切包名（C++ 包是 `gcc-toolset-14-gcc-c++` 还是其它命名），以确保替换名有效。
- 确认 `gcc-toolset-14-gcc*` 通配符是否确实包含 gcc-c++（若包含，则该行可安全删除；若不包含，则需显式改为 `gcc-toolset-14-gcc-c++`）。
- 确认 aarch64 架构 job 的日志（本次提供的是 x86-64 路径日志）是否同样在该行失败，以判断是否为跨架构一致问题。

## 修复验证要求
本修复方向不涉及"修改正则匹配第三方/上游源文件"，无需按该条要求从上游拉取文件校验。但 code-fixer 在提交前应验证：修正后的 yum 包名在 openEuler 24.03-LTS-SP4 仓库中确实可被匹配（例如以基础镜像 `openeuler/openeuler:24.03-lts-sp4` 执行 `yum list available 'gcc-toolset-14*'` 确认包清单），避免再次出现 `Unable to find a match`。

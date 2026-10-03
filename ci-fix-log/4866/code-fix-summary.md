# 修复摘要

## 修复的问题
修正新增 Dockerfile 中 openEuler 24.03-LTS-SP4 不存在的 C++ 编译器包名 `gcc-toolset-14-c++*`，改为实际存在的 `gcc-toolset-14-gcc-c++*`，使构建阶段的 `yum install` 不再报错。

## 修改的文件
- `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile`: 将 `yum install` 列表中的 `gcc-toolset-14-c++*` 改为 `gcc-toolset-14-gcc-c++*`。

## 修复逻辑
分析报告的候选根因因缺少日志而无法确认，本次先从 PR 门禁评论中取得真实失败 job 日志：

- x86_64 check_build #4980：`No match for argument: gcc-toolset-14-c++*` / `Error: Unable to find a match: gcc-toolset-14-c++*`
- aarch64 check_build #5076：同样报错 `Error: Unable to find a match: gcc-toolset-14-c++*`

两个架构都精确失败在 Dockerfile 第 8 行的 `RUN yum update -y && yum install ...` 步骤。

对照 openEuler 官方仓库 `everything` 源核实包名：
- 24.03-LTS-SP4（x86_64 与 aarch64）中 C++ 编译器包为 `gcc-toolset-14-gcc-c++-14.3.1-18.oe2403sp4`，不存在任何 `gcc-toolset-14-c++*` 包。
- 24.03-LTS-SP2（旧版 1.22.1 Dockerfile 所基于的版本）中确实存在 `gcc-toolset-14-c++-14.2.1-8.oe2403sp2`，因此同一写法在 sp2 能通过、在 sp4 会失败，属于基础镜像版本升级导致的包名变化（`c++` → `gcc-c++`）。

同时核实 `microsoft/onnxruntime` 上游存在 `v1.30.0` tag（已通过 GitHub API 确认），故分析报告候选根因 1（上游 tag 不存在）不成立，无需修改 `ARG VERSION`。

报告候选根因 3（gitee/atomgit 链接、末行换行）经核对：仓库中 `gitee.com/openeuler/openeuler-docker-images` 链接为多数既有用法，且该差异不会导致 check_build 失败，故不做改动（不扩展范围）。

## 潜在风险
- 该改动仅涉及包名修正，`gcc-toolset-14-gcc*` 通配本身也能匹配 `gcc-toolset-14-gcc-c++`，改后显式安装同一 C++ 编译器包，不影响其它依赖；无功能性风险。
- `yum install` 通过后，后续 `./build.sh` 编译阶段是否会暴露其他版本适配问题，需由 CI 重新构建验证；本次仅修复已由日志证实的第一处 error，未做推断性改动。
# 修复摘要

## 修复的问题
将 `cmake==3.28` 修正为 PyPI 上真实存在的 `cmake==3.28.4`，解决构建阶段 `No matching distribution found for cmake==3.28` 的失败。

## 修改的文件
- `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile`: 第 29 行 `cmake==3.28` → `cmake==3.28.4`

## 修复逻辑
1. 分析报告本身未附带 `ci.logs`（由于日志抓取正则只匹配 `ci.openeuler.openatom.cn`，而 CI 机器人现已改为评论 `log-ci.openeuler.openatom.cn`，导致分析阶段取不到日志）。为取得真实根因，本次直接从 GitCode API 拉取了修复 PR #4878 的门禁结果，并通过 openEuler Jenkins 日志接口获取了失败构建的完整日志：
   - x86_64：`multiarch/openeuler/x86-64/openeuler-docker-images` build #4993
   - aarch64：`multiarch/openeuler/aarch64/openeuler-docker-images` build #5089
2. 日志显示上一轮修复（`gcc-toolset-14-c++*` → `gcc-toolset-14-gcc-c++*`）已生效：builder 阶段 yum 安装成功（`gcc-toolset-14-gcc-c++-14.3.1-18.oe2403sp4` 已安装），构建推进到下一步。
3. 随后在 `#8 [builder 3/4] RUN pip install ... cmake==3.28` 处出现第一条真实错误：
   ```
   ERROR: Could not find a version that satisfies the requirement cmake==3.28
   ERROR: No matching distribution found for cmake==3.28
   ```
   两个架构日志一致。
4. 经核对 PyPI `cmake` 版本，`3.28` / `3.28.0` 均不存在，3.28 系列仅有 `3.28.1`、`3.28.3`、`3.28.4`。上游 `microsoft/onnxruntime` v1.30.0 的 `cmake/CMakeLists.txt` 要求 `cmake_minimum_required(VERSION 3.28)`，因此选用 3.28 系列最新补丁 `3.28.4`，既满足最低版本要求，又保持原意（尽量贴近 3.28）。
5. 已从 PyPI 验证 `cmake==3.28.4` 提供 `manylinux2014_x86_64` 与 `manylinux2014_aarch64` wheel，可覆盖 amd64 / arm64 两种构建架构。

## 潜在风险
无。改动仅改小版本号，且 3.28.4 满足 onnxruntime v1.30.0 的 CMake 最低版本约束；不影响其他文件或功能。
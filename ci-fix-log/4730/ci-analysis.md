# CI 失败分析报告

## 基本信息
- PR: #4730 — 【自动升级】onnxruntime容器镜像升级至1.30.0版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 工具链包名不存在
- 新模式症状关键词: No match for argument, yum install, gcc-toolset-14-c++, Unable to find a match

## 根因分析

### 直接错误
```
#7 52.65 No match for argument: gcc-toolset-14-c++*
#7 52.69 Error: Unable to find a match: gcc-toolset-14-c++*
#7 ERROR: process "/bin/sh -c yum update -y &&     yum install -y         tar         ca-certificates         gcc-toolset-14-gcc*         gcc-toolset-14-binutils*         gcc-toolset-14-c++*         ..." did not complete successfully: exit code: 1
Dockerfile:12
...
ERROR: failed to solve: process "... gcc-toolset-14-c++* ..." did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile:8-25`（`RUN yum install ...` 步骤），具体为第 14 行的 `gcc-toolset-14-c++*`
- 失败原因: 新增 Dockerfile 的 builder 阶段在 openEuler 24.03-LTS-SP4 仓库中安装 `gcc-toolset-14-c++*`，yum 匹配不到任何包（`No match for argument`），整个 `yum install` 事务失败，Docker 构建以 exit code 1 终止。

### 与 PR 变更的关联
直接相关。本 PR 新增了 `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile`，其第一个 `RUN yum install` 步骤（第 8-25 行）中包含不存在的包名模式 `gcc-toolset-14-c++*`。CI 日志开头的 `Difference: ["AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile", ...]` 也确认构建对象正是该新增文件。日志中 `gcc-toolset-14-gcc*`、`gcc-toolset-14-binutils*` 未报未匹配，仅 `gcc-toolset-14-c++*` 报 `No match`，说明失败精确锁定在该包名上（非网络或仓库整体不可用）。

## 修复方向

### 方向 1（置信度: 高）
去掉或修正无效的 `gcc-toolset-14-c++*` 包名。openEuler 的 gcc-toolset 工具链中 C++ 编译器通常作为 `gcc-toolset-14-gcc-c++`（或已由 `gcc-toolset-14-gcc*` 通配匹配包含）提供，`gcc-toolset-14-c++` 这一独立包名并不存在。应改为仓库中实际存在的包名，或直接移除该行（通过通配 `gcc-toolset-14-gcc*` 获得 C++ 编译器）。

### 方向 2（可选）
对照仓库中已有同类镜像（如 `AI/onnxruntime/1.22.1/24.03-lts-sp2/Dockerfile`）的工具链安装写法，沿用其已验证可用的包名/通配方式，避免自造包名。

## 需要进一步确认的点
- 在 openEuler 24.03-LTS-SP4 仓库中确认 `gcc-toolset-14` 系列的实际包名（`gcc-toolset-14-gcc-c++`、`gcc-toolset-14-gcc`、`gcc-toolset-14-binutils` 等），以确定 C++ 编译器由哪个包/通配提供。
- 确认 `gcc-toolset-14` 是否确实在 SP4 的 `everything`/`update` 等仓库中发布（日志仅显示 c++ 未匹配，其余通配未报错，倾向已发布）。
- 对照 `AI/onnxruntime/1.22.1/24.03-lts-sp2/Dockerfile` 中工具链安装行，确认历史可用的包名写法。

## 修复验证要求
本修复为 Dockerfile 内包名修正，不涉及正则 patch 外部源文件，无需上游文件验证。但建议 code-fixer 在提交前通过 `yum list available 'gcc-toolset-14*'`（或等效方式）确认所改包名在 openEuler 24.03-LTS-SP4 仓库中确实存在。

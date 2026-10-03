# 修复摘要

## 修复的问题
无需修改源代码。经获取真实 CI 日志核实，本 PR 的唯一失败项 `aarch64 check_build` 属于 CI 基础设施/运行环境问题（aarch64 构建节点缺少 `eulerpublisher` 可执行文件），grafana 13.2.3 镜像本身已成功构建并推送，与 PR 修改的文件无关。

## 修改的文件
- 无（未修改任何源文件，保持 PR 原始内容不变）

## 修复逻辑

### 真实 CI 结果（来自 PR #4859 的 openeuler-bot 评论，2026-10-03T15:25）
| Check Name | Build Result |
|---|---|
| check_package_license | ⚠ WARNING（非失败：缺少项目级 Copyright 声明文件） |
| check_sca | ✅ SUCCESS |
| x86_64 `check_build` | ✅ SUCCESS |
| aarch64 `check_build` | ❌ FAILED |

- 触发层日志（job 5202）：`Finished: SUCCESS`；
- x86_64 构建日志（job 4973）：`Finished: SUCCESS`，镜像 push 成功并通过 `[Check]` 镜像测试；
- aarch64 构建日志（job 5069）：镜像 **构建成功**（`#9 DONE 148.5s`、`Complete!`）、**推送成功**（`#10 DONE 104.2s`、`[Build] finished` / `[Push] finished`），失败发生在推送之后的 CI 自有后处理步骤：
  ```
  File ".../eulerpublisher/update/container/app/update.py", line 256, in check_updates
    if _check_app_image(file=file) != 0:
  ...
  FileNotFoundError: [Errno 2] No such file or directory: 'eulerpublisher'
  Build step 'Execute shell' marked build as failure
  ```

### 各可疑点逐一排除
1. **许可/Copyright（分析报告可疑点 A）**：触发层日志明确显示仓库 license 检查 `pass`，copyright 缺失仅为 `WARNING`（且为仓库级既有问题，非本 PR 引入），未导致门禁失败。grafana 目录下 32 个 Dockerfile 均无 SPDX/Copyright 头，属仓库既有约定，不构成本次失败原因，且修复被禁止新增文件。
2. **Dockerfile 反斜杠尾随空格（分析报告可疑点 B）**：日志显示该 RUN 指令被正确解析，x86_64 与 aarch64 均成功执行安装并构建出镜像；且 13.2.2 基线存在完全相同写法。非失败原因。
3. **RPM 下载可用性（分析报告方向 3）**：已实测 `https://dl.grafana.com/enterprise/release/grafana-enterprise-13.2.3-1.x86_64.rpm` 与 `...-1.aarch64.rpm` 均返回 HTTP 200；aarch64 日志中实际成功下载 `grafana-enterprise-13.2.3-1.aarch64.rpm`。非失败原因。

### 结论
失败根因为 CI 基础设施问题（aarch64 构建节点 `eulerpublisher` 命令缺失），非源码缺陷。按流程约定，infra-error 情况不做代码修改，避免引入无关改动。

## 潜在风险
无。未改动任何源文件，不会影响镜像构建或既有功能。建议在基础设施侧修复 aarch64 节点（确保 `eulerpublisher` 可执行文件在 PATH 中）后重跑门禁。
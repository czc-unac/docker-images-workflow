# 修复摘要

## 修复的问题
新增的 3dslicer 5.12.4 构建脚本缺少可执行权限，导致 Dockerfile 以 `./build-*.sh` 调用时返回 `Permission denied`（exit code 126）。

## 修改的文件
- `HPC/3dslicer/5.12.4/24.03-lts-sp4/build-CTKAppLauncher.sh`: 恢复可执行权限（git 模式 100644 → 100755）
- `HPC/3dslicer/5.12.4/24.03-lts-sp4/build-tbb.sh`: 恢复可执行权限（git 模式 100644 → 100755）
- `HPC/3dslicer/5.12.4/24.03-lts-sp4/build-Slicer.sh`: 恢复可执行权限（git 模式 100644 → 100755）

## 修复逻辑
分析报告根因定位为 `Dockerfile:21` 直接以 `./build-CTKAppLauncher.sh` 方式执行脚本，而 PR 新增的三个脚本 git 模式为 `100644`（不可执行），首个脚本即失败。对照同仓库既有可用版本 `HPC/3dslicer/5.8.1/24.03-lts-sp1/`，其三个同名脚本均为 `100755`，且 Dockerfile 调用方式完全一致。因此本修复采用报告中的方向 1：用 `git update-index --chmod=+x` 将三个脚本在索引中设为 `100755`，并同步 `chmod +x` 工作区文件，使提交后保留 Unix 可执行位。修复后 `git ls-files -s` 已确认三者均为 `100755`，与 5.8.1 版本配置一致，未改动 Dockerfile 及调用逻辑，符合最小化原则。

## 潜在风险
无。仅恢复脚本可执行位，不改变脚本内容与 Dockerfile 调用方式，与仓库既有 5.8.1 版本行为一致。分析报告提及的 `zlib.patch` 兼容性属后续潜在风险，非本次失败根因，未做改动。
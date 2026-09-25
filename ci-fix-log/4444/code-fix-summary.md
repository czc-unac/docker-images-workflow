# 修复摘要

## 修复的问题
修复新增的 milc `6b9b8a0` 镜像 Dockerfile 因把 commit 短哈希当作分支名传给 `git clone --branch` 而导致 `fatal: Remote branch 6b9b8a0 not found in upstream origin`（exit 128）的构建失败。

## 修改的文件
- `HPC/milc/6b9b8a0/24.03-lts-sp4/Dockerfile`: 第 18-20 行，去掉 `--depth 1 --branch 6b9b8a0`，改为正常克隆后再 `git checkout ${VERSION}`。

## 修复逻辑
- 分析报告根因：`6b9b8a0` 是上游 `milc-qcd/milc_qcd` 的 **commit 短哈希**，而非分支/tag；`git clone --branch <sha>` 会在远端查找同名 ref 失败。
- 修复采用分析报告的「方向 1」：先完整克隆仓库，再 `git checkout ${VERSION}`。`git checkout <短哈希>` 可在克隆下来的完整历史中解析出对应 commit，从而正确取得该版本源码。
- 使用 `${VERSION}` 变量（已在第 4 行定义为 `6b9b8a0`）而非硬编码哈希，保持与其它版本 Dockerfile 的结构一致，且该写法对分支名/tag/commit 均有效，便于自动升级流程复用。

## 上游验证结果（提交前已实测）
- 通过 `git ls-remote https://github.com/milc-qcd/milc_qcd.git` 确认：`6b9b8a06eec5746187bbfd197eac2629ab8d8e72` 为上游真实 commit（同时是 `refs/heads/develop` 与 HEAD 指向的提交）。
- 实测发现 GitHub **不支持**按短哈希 fetch：`git fetch --depth 1 origin 6b9b8a0` 报 `couldn't find remote ref 6b9b8a0`；按完整 SHA fetch 可成功，但 Dockerfile 中只有短哈希。因此在 `--depth 1` 浅克隆下无法可靠取得任意短哈希 commit，去掉浅克隆是必要的。
- 按修改后 Dockerfile 的等价命令序列实测（`git clone ... && git checkout 6b9b8a0`）：克隆成功，checkout 后 `git rev-parse --short HEAD` 得到 `6b9b8a06`，且 `CMakeLists.txt` 存在，验证通过。上游仓库无 submodule，无需额外初始化。
- 本修复不涉及正则 patch 外部源文件。

## 潜在风险
- 去掉了 `--depth 1`，克隆会拉取完整历史，镜像构建的网络传输量与时间略增（实测全量克隆约 6 秒，相对于后续 MILC cmake 编译耗时可忽略）。
- 若未来自动升级传入的 `VERSION` 既非有效 commit 也非有效 ref，`git checkout ${VERSION}` 仍会失败并中断构建；但这是无法解析的版本标识本身的错误，属于预期行为，不应由此修复掩盖。
- 已确认上游无 submodule，故未添加 `--recurse-submodules`，不影响构建。
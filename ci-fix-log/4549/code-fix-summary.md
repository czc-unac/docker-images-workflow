# 修复摘要

## 修复的问题
新增的 3 个构建脚本缺少可执行位，导致 Dockerfile 中 `RUN ./build-CTKAppLauncher.sh ...` 以直接执行方式调用时返回 `Permission denied`（退出码 126），构建在第一步即终止。

## 修改的文件
- `HPC/3dslicer/5.12.4/24.03-lts-sp4/build-CTKAppLauncher.sh`: 文件模式 `100644` → `100755`（补充可执行位）
- `HPC/3dslicer/5.12.4/24.03-lts-sp4/build-tbb.sh`: 文件模式 `100644` → `100755`（补充可执行位）
- `HPC/3dslicer/5.12.4/24.03-lts-sp4/build-Slicer.sh`: 文件模式 `100644` → `100755`（补充可执行位）

改动仅为文件模式，未改动任何文件内容。修复后 `git ls-files -s` 显示三个脚本均为 `100755`。

## 修复逻辑
分析报告指向的根因是：`COPY` 进镜像的脚本不具备可执行位，而 Dockerfile 第 21-23 行直接以 `./build-*.sh` 方式调用，因此 `/bin/sh` 报 `Permission denied`（126）。这与日志完全吻合。

采用分析报告的**方向 1（仓库侧最根本的修法）**，恢复三个脚本的可执行位。对照证据：
- 本仓库已存在的同组件旧版本 `HPC/3dslicer/5.8.1/24.03-lts-sp1/` 中，这三个脚本的 git 模式均为 `100755`；
- 且本次新增的脚本内容与旧版本**逐字节相同**（blob 哈希一致：build-CTKAppLauncher.sh=`03652f1`、build-Slicer.sh=`387648f`、build-tbb.sh=`f72bc92`），说明内容无问题，唯一回归就是文件模式从 `100755` 退化为 `100644`。

因此恢复模式即可修复该 CI 失败，无需改动 Dockerfile 或脚本内容，改动范围最小。`zlib.patch` 无需执行权限，保持 `100644` 不变。

## 潜在风险
- **后续 `zlib.patch` 应用可能失败（建议下一步 CI 关注）**：分析报告已提示需确认 `zlib.patch` 对上游 `External_zlib.cmake` 的 hunk 是否匹配。已从上游 Slicer 仓库 `v5.12.4` tag 获取 `SuperBuild/External_zlib.cmake`，并在临时 git 仓库中执行 `git apply --check` 实测，结果为 **patch does not apply**。原因是上游 5.12.4 已改变该文件（`-DZLIB_MANGLE_PREFIX` 改为 `-DZLIB_SYMBOL_PREFIX`，并新增 `-DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=ON`），patch 第 2 个 hunk 的上下文不再匹配。权限修复后构建将走到 `build-Slicer.sh` 的 `git apply zlib.patch`，预计会在该处失败。
  - 本次未修改 `zlib.patch`：一是它尚未在 CI 中实际触发（当前失败止于权限步骤），分析报告亦将该点列为"权限修复后重新触发 CI 验证"的待确认项；二是修正 patch 需要判断 5.12.4 上游已内置 PIC 后该 patch 是否仍必要，属于新的问题范围。建议下一轮 CI/修复迭代针对该 patch 做最小化更新（例如按 5.12.4 上下文重新生成 patch，或确认上游已内置 `CMAKE_POSITION_INDEPENDENT_CODE` 后移除 patch 应用）。
- 其余文件（Dockerfile、zlib.patch、README.md、doc/image-info.yml、meta.yml、build-* 脚本内容）均未改动，不影响其他功能。
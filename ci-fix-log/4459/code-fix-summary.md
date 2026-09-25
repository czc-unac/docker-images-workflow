# 修复摘要

## 修复的问题
`Others/bcache/1.0.8/24.03-lts-sp4/Dockerfile` 中 `git clone` 构造出的 ref `bcache-tools-1.0.8` 在上游 `colyli/bcache-tools` 仓库不存在，导致构建在第 5/7 步 clone 时报 `fatal: Remote branch ... not found`（exit 128）。

## 修改的文件
- `Others/bcache/1.0.8/24.03-lts-sp4/Dockerfile`: 将 `git clone` 的分支参数由 `bcache-tools-${VERSION}`（展开为不存在的 `bcache-tools-1.0.8`）改为上游实际存在的 tag `bcache-tools-1.1`，克隆目标目录仍为 `/opt/bcache-tools-${VERSION}`，后续 `WORKDIR`、`patch`、`make install` 流程不变。

## 修复逻辑
1. 通过 `git ls-remote --tags https://git.kernel.org/pub/scm/linux/kernel/git/colyli/bcache-tools.git` 核实：该上游仓库当前只有唯一的 tag `bcache-tools-1.1`（commit `62f7b909`），不存在 `bcache-tools-1.0.8`、`1.0.8`、`v1.0.8` 等任何 ref。这正是 CI 报 `Remote branch not found` 的根因。
2. 分析报告匹配知识库「模式22：Git分支名构造错误」，其历史修复（PR #2653 CMAQ）的做法是将 clone 的分支名直接改为经 `git ls-remote` 验证存在的上游 tag；本修复沿用同一思路，将 ref 修正为已验证存在的 `bcache-tools-1.1`。
3. 保持 `ARG VERSION=1.0.8` 不变，仅修正 clone 行，改动范围最小（单行）。
4. 新增的 `Export-CACHED_UUID-and-CACHED_LABEL.patch` 与 1.1/24.03-lts-sp4 目录下的 patch 字节完全一致（此前已 `diff` 确认），是面向 1.1 源码生成的；改用 1.1 源码后 patch 可正常应用，避免了分析报告「需进一步确认点 2」中提到的 hunk 偏移问题。
5. 已在本机用 `docker build`（基础镜像 `openeuler/openeuler:24.03-lts-sp4`）完整验证：clone、patch、`make -j`、`make install` 全部成功，产物 `/usr/sbin/make-bcache`、`/usr/sbin/bcache-super-show`、`/lib/udev/bcache-export-cached` 均正常安装。

## 潜在风险
- 上游 colyli/bcache-tools 已无 1.0.8 tag，当前唯一可用源码为 1.1。因此该镜像实际打包的是 bcache-tools 1.1，而目录名、`meta.yml`/`README.md`/`image-info.yml` 中的 tag 仍为 `1.0.8-oe2403sp4`，存在版本语义不一致（与知识库中 mongoose #2731 的既有修复情况相同）。若上游未来恢复 1.0.8 tag，或需发布真正的 1.0.8 镜像，需另行提供与该版本源码匹配的 patch（真实 1.0.8 源码在当前 GCC 下还存在 `crc64` inline 链接问题）。
- 该修复不改动 `README.md`、`doc/image-info.yml`、`meta.yml` 及 patch 文件，不引入额外范围改动。
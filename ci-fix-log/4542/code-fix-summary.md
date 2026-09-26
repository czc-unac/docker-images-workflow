# 修复摘要

## 修复的问题
修复 bcache 1.0.8 容器镜像构建失败：上游 kernel.org 仓库不存在 `bcache-tools-1.0.8` ref，`git clone --branch bcache-tools-1.0.8` 报 `Remote branch not found`（exit 128）；同时 1.0.8 老源码与随附 patch、现代 GCC 均不兼容。

## 修改的文件
- `Others/bcache/1.0.8/24.03-lts-sp4/Dockerfile`:
  - 新增 `ARG COMMIT=a73679b22c333763597d39c72112ef5a53f55419`（bcache-tools 1.0.8 对应的上游发布提交）。
  - 将 `git clone --depth 1 --branch bcache-tools-${VERSION} ...` 改为完整 `git clone` 后 `git checkout ${COMMIT}`，不再依赖不存在的同名 tag。
  - `make` 步骤增加 `CFLAGS="-fgnu89-inline"`，规避 1.0.8 源码中 `inline uint64_t crc64()` 在 C99/C11 inline 语义下不生成外部符号导致的链接错误 `undefined reference to crc64`。
- `Others/bcache/1.0.8/24.03-lts-sp4/Export-CACHED_UUID-and-CACHED_LABEL.patch`:
  - 修正 Makefile hunk 的上下文行：1.0.8 的 Makefile 安装行没有 `bcache` 这一目标（该目标在 1.1 才引入），原先照搬 1.1 的 patch 上下文导致 `Hunk #1 FAILED`。去掉多余的 `bcache ` 后 hunk 可正常应用。

## 修复逻辑
1. **根因（clone ref 不存在）**：通过 `git ls-remote https://git.kernel.org/pub/scm/linux/kernel/git/colyli/bcache-tools.git` 确认上游实际只存在 `refs/heads/master`、`refs/heads/nvdimm_meta`、`refs/heads/zonde-device`、`refs/tags/bcache-tools-1.1`，**不存在任何 1.0.8 命名的 ref**，因此分析报告方向 1（改 ref 名）不可行。
2. **确定 1.0.8 真实来源**：bcache-tools 1.0.8 是 2014-12-04 的历史发布。已从上游仓库历史中定位发布提交 `a73679b22c333763597d39c72112ef5a53f55419`，并将其与 Debian/Ubuntu 的 `bcache-tools_1.0.8.orig.tar.gz` 逐文件 `diff -rq` 比对，内容完全一致，确认该提交即真正的 1.0.8 源码。故改用「完整 clone + checkout 提交」获取，避免 1.0.8 对应 ref 缺失以及 `git.kernel.org` snapshot 对不存在 tag 返回 HTML 的问题（模式32）。
3. **patch 兼容性**：在内存中针对 1.0.8 源码实测 patch，唯一失败 hunk 为 `Makefile`（上下文 `... bcache-super-show\tbcache $(DESTDIR)...` 中的 `bcache` 只在 1.1 存在）。修正该上下文后 `patch -p1 --dry-run` 与实跑均 `exit 0`，且 69-bcache.rules、bcache-export-cached、initcpio/install、initramfs/hook 各 hunk 均正常应用。
4. **编译兼容性**：1.0.8 的 `bcache.c` 使用 `inline uint64_t crc64()`（无 `static`、无外部定义），现代 GCC 默认 gnu11/gnu17 下不会生成外部符号，链接 `make-bcache`/`bcache-super-show` 时 `undefined reference to crc64`。使用 `-fgnu89-inline`（GNU89 inline 语义会生成符号）修复。

## 验证结果
- `git ls-remote` 已确认上游 ref 列表（无 1.0.8）。
- 已从上游获取实际源码提交并与 1.0.8 官方 orig tarball 逐文件比对一致。
- 已按 Dockerfile 的完整流程在本地模拟：`git clone` → `git checkout a73679b` → 应用仓库内 patch → `CFLAGS="-fgnu89-inline" make -j` → `make install DESTDIR=...`，构建与安装全部成功，产物包含 make-bcache、bcache-super-show、probe-bcache、bcache-register、bcache-export-cached、69-bcache.rules 等。

## 潜在风险
- 硬编码了上游 1.0.8 发布提交哈希 `a73679b...`；该提交为历史提交，稳定存在且可被完整克隆获取，但不会随上游更新。
- `-fgnu89-inline` 会改变整个编译单元的 inline 语义；本项目代码量小且已实测编译/链接通过，但若未来升级源码需复评。
- patch 文件针对 1.0.8 的 Makefile 上下文做了专用调整，与 `Others/bcache/1.1/...` 下的同名 patch 不再完全一致，后续维护需按版本区分。
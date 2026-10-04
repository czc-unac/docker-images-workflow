# 修复摘要

## 修复的问题
jetty 自动升级 PR 指向了 Maven Central 上尚不存在的版本 `12.1.14`，导致 `jetty-home-12.1.14.tar.gz` 下载 404、镜像构建失败；已回退到 Maven Central 上真实存在的最新版本 `12.1.13`。

## 修改的文件
- `Others/jetty/12.1.14/24.03-lts-sp4/Dockerfile`: 第 3 行 `ARG VERSION=12.1.14` 改为 `ARG VERSION=12.1.13`。
- 其余 PR 文件（`docker-entrypoint.sh`、`generate-jetty-start.sh`、`README.md`、`doc/image-info.yml`、`meta.yml`）保持原样不动。

## 修复逻辑
分析报告（模式42，证据不足）指向 `Others/jetty/12.1.14/24.03-lts-sp4/Dockerfile` 中由 `ARG VERSION` 驱动的 Maven Central 下载地址。本次独立验证确认了根因：

1. **拉取真实 CI 日志**：通过 GitCode PR #4897 评论中的 Jenkins 构建链接获取到 aarch64 构建 job 日志（`/job/multiarch/openeuler/aarch64/openeuler-docker-images/5112/`），构建在 `[3/6] RUN ... curl -SL .../jetty-home-12.1.14/jetty-home-12.1.14.tar.gz ... tar -xvf ...` 层失败：
   ```
   gzip: stdin: not in gzip format
   tar: Child returned status 1
   tar: Error is not recoverable: exiting now
   sed: can't read etc/jetty.conf: No such file or directory
   ERROR: process ... did not complete successfully: exit code: 1
   ```
2. **排除“缺少 curl”假设（分析报告方向2）**：实际拉取并检查基础镜像 `openeuler/openeuler:24.03-lts-sp4`，镜像内自带 `curl 8.4.0`，因此不是缺少 curl 的问题。
3. **确认版本不存在**：直接请求 `https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-home/12.1.14/jetty-home-12.1.14.tar.gz` 返回 **HTTP 404**，且 `maven-metadata.xml` 中 `<latest>` 为 `12.1.13`（不含 12.1.14）；而 `12.1.13` 的 tarball 返回 200。该问题与知识库模式42 中的历史案例 PR #4852（同路径、同版本 `12.1.14`）完全一致。注：上游 GitHub 虽在 2026-10-02 打了 `jetty-12.1.14` tag，但该版本尚未发布到 Maven Central，故 Dockerfile 所依赖的下载地址必然 404 并引发 `tar` 解包失败。
4. **实施最小修复**：仅将 `ARG VERSION` 改为 Maven Central 可用的最新版本 `12.1.13`，即可让下载与 `tar` 解包成功、构建通过。

**验证结果**：修改后在基础镜像中实际执行下载与解包，`curl` 得到 46,110,571 字节的有效 gzip，`tar -tzf` 成功（`TAR_OK`）。修改后的 Dockerfile 内容与同类问题已通过 CI 的参考修复（fix PR #4880，标签 `ci_successful`）逐字节一致。

## 潜在风险
- 镜像 Tag 仍为 `12.1.14-oe2403sp4`，但镜像内实际 Jetty 版本为 `12.1.13`（这是自动升级 PR 的固有属性，参考修复 #4880 采用同样处理并已通过 CI）。待上游 `12.1.14` 正式发布到 Maven Central 后，可再将其更新为 `12.1.14`。
- 未改动 README/image-info/meta，保持与参考修复一致的登记方式，不影响其他镜像。
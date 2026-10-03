# 修复摘要

## 修复的问题
jetty 自动升级 PR 指向了上游不存在的版本 `12.1.14`，导致下载 `jetty-home-12.1.14.tar.gz` 返回 404、镜像构建失败；已回退到 Maven Central 上真实存在的最新版本 `12.1.13`。

## 修改的文件
- `Others/jetty/12.1.14/24.03-lts-sp4/Dockerfile`: 第 3 行 `ARG VERSION=12.1.14` 改为 `ARG VERSION=12.1.13`。

## 修复逻辑

### 根因（已经真实 CI 日志与上游制品双重证实）
分析报告置信度为「低」且仅基于 diff 静态推断（怀疑缺少 `curl`）。本次修复**未采用该推断**，而是通过真实证据重新定位根因：

1. 通过 GitCode API 读取 PR #4852 的门禁结果评论，得到真实失败 job：
   - x86_64 `check_build` FAILED（Jenkins build #4966）
   - aarch64 `check_build` FAILED（Jenkins build #5062）
   - `check_package_license` 仅为 WARNING，`check_sca` SUCCESS，均非失败原因。
2. 拉取真实构建日志（`https://log-ci.openeuler.openatom.cn/api/build/log/download?job=multiarch/openeuler/x86-64/openeuler-docker-images&build=4966`），关键错误为：
   ```
   0   4990      0 --:--:-- --:--:--  5036
   0.272 gzip: stdin: not in gzip format
   0.272 tar: Child returned status 1
   0.272 tar: Error is not recoverable: exiting now
   0.273 sed: can't read etc/jetty.conf: No such file or directory
   0.275 Error: Unable to access jarfile /usr/local/jetty/start.jar
   ```
   `curl` 正常执行（没有 `command not found`），但只下载到 5036 字节的 HTML 内容，`tar` 报 “not in gzip format”。因此分析报告「缺少 curl 依赖」的方向被排除。
3. 校验上游制品：
   - `https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-home/12.1.14/jetty-home-12.1.14.tar.gz` → HTTP 404（Maven Central 无此版本；`maven-metadata.xml` 的 `<latest>/<release>` 为 12.1.13，目录列表最高只到 12.1.13）。
   - `.../12.1.13/jetty-home-12.1.13.tar.gz` → HTTP 200。
   即 GitHub 上存在 tag `jetty-12.1.14`，但对应 Maven 制品从未发布，自动升级工具误将 tag 当作可构建版本。

### 修复方式依据
仅将下载版本改回真实可用的 `12.1.13`，保持其余文件与目录不变（`meta.yml` / `README.md` / `image-info.yml` 中 12.1.14 的条目未改动）。该做法与本仓库已有的同类自动修复先例一致：`Others/mongoose/7.22/24.03-lts-sp3/Dockerfile`（PR #2731，commit `845931004`）中，上游 tag 不存在时同样只把 `ARG VERSION=7.22` 改为 `7.21`，目录名与元数据保持不变；仓库历史中多个 `fix(ci): 修复摘要` 提交也采用“把 `ARG VERSION` 改回有效版本”的最小化修复模式。

未修改 `meta.yml` / `README.md` / `image-info.yml`，因为原修复的最小单元只涉及 Dockerfile 的版本号，且修改元数据会扩大改动范围、触发额外的版本一致性争议。

### 验证结果
- 已确认 `12.1.13` 制品可下载（HTTP 200），且仓库中已存在同结构的 `12.1.13/24.03-lts-sp4` 镜像（其 `--add-to-start` 模块列表与本文件完全一致，构建历史通过）。
- 本次修复为单行改动，不涉及正则 patch 外部源文件。

## 潜在风险
- 目录名与镜像标签仍为 `12.1.14`，但实际构建内容为 `12.1.13`；这会带来版本标识与实际制品不一致的问题。这是遵循本次“仅修改 PR 已涉及文件、不新增/删除文件”约束下的最小化修复，与仓库既有同类修复先例保持一致。根治需要上游正式发布 12.1.14 制品后再重新升级。
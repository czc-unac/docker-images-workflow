# 修复摘要

## 修复的问题
无需代码修改。经获取真实 CI 日志确认：本次失败根因是上游 Eclipse Jetty 的预编译发行包 `jetty-home-12.1.14.tar.gz` 尚未发布到 Maven 中央仓（HTTP 404），`curl -SL` 将 554 字节的 404 页面存入 `jetty.tar.gz`，随后 `tar -xvf` 报 `gzip: stdin: not in gzip format` 而构建失败。这属于上游制品尚未产出/发布的时序问题，在 `pr.changed_files` 允许的文件范围内不存在既能修复构建、又能保留“升级到 12.1.14”语义的最小改动，故本次不做任何代码改动。

## 修改的文件
- 无（未修改任何源文件）

## 修复逻辑

### 1. 获取真实 CI 日志（不再依据“日志缺失”的假设）
入参 `ci_analysis` 标注日志缺失、置信度低。为获得确定性证据，从 PR #4852 的门禁评论表格定位到失败 job，并直接下载控制台日志：
- x86_64: `https://ci.openeuler.openatom.cn/job/multiarch/job/openeuler/job/x86-64/job/openeuler-docker-images/4966/consoleText`
- aarch64: `https://ci.openeuler.openatom.cn/job/multiarch/job/openeuler/job/aarch64/job/openeuler-docker-images/5062/consoleText`

两个架构失败完全一致，失败发生在 `Dockerfile:17` 的 `RUN` 步骤（`[3/6]`）：
```
#9 [3/6] RUN ... curl -SL https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-home/12.1.14/jetty-home-12.1.14.tar.gz -o jetty.tar.gz ; tar -xvf jetty.tar.gz --strip-components=1 ; ...
#9 0.156   % Total ... 100   554    0   554 ...
#9 0.272 gzip: stdin: not in gzip format
#9 0.272 tar: Child returned status 1
#9 0.272 tar: Error is not recoverable: exiting now
#9 0.273 sed: can't read etc/jetty.conf: No such file or directory
#9 ERROR: process "/bin/sh -c mkdir -p ..." did not complete successfully: exit code: 1
```
`curl` 只下载到 554 字节（404 错误页），因此 `tar` 解压失败，后续 `start.jar` 步骤全部连锁失败。

### 2. 排除分析报告中的怀疑点 #1（缺少 curl）
日志显示基础镜像已自带并升级了 curl（`curl 8.4.0-37.oe2403sp4 update`），且该 `curl` 命令实际执行并写入了文件，因此“未安装 curl”不是根因。

### 3. 确认根因：上游仓库不存在预编译制品（模式02：软件包版本不存在）
以 Dockerfile 中 `ARG VERSION=12.1.14` 为准，对多个来源做了确定性验证：
- `https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-home/12.1.14/jetty-home-12.1.14.tar.gz` → **HTTP 404**（响应头 `x-amz-error-code: NoSuchKey`，确为对象不存在而非网络问题）
- `https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-home/maven-metadata.xml` → 最新版本为 **12.1.13**（`lastUpdated: 20260907`）
- Jetty 官方下载页 `https://jetty.org/download.html` → 稳定版仍为 **12.1.13**
- GitHub `jetty/jetty.project` releases/latest → **12.1.13**；`releases/tags/jetty-12.1.14` → 404；仅存在 git tag `jetty-12.1.14`（创建于 2026-10-02，无可下载 release 制品）
- 镜像站 `maven.aliyun.com`、Sonatype releases、Eclipse download 均无 12.1.14 制品

即：上游已打 tag 但尚未完成 Maven 中央仓发布，属发布时序问题。

### 4. 为什么不做代码改动
- 现有所有 jetty 版本的下载源均为 `repo1.maven.org` 上的 `jetty-home` 预编译发行包；当前无任何可用来源提供 12.1.14 的该制品，更换下载源无法解决。
- 若将 `ARG VERSION` 降回可用的 12.1.13，会与目录名 `12.1.14/`、`meta.yml` 的 `12.1.14-oe2403sp4`、`README.md`/`image-info.yml` 的 tag 及 PR 标题“升级至 12.1.14”冲突，并可能与既有 12.1.13 条目重复；这既不满足最小修复原则，也会产生“tag 为 12.1.14、内容为 12.1.13”的错误镜像。在只允许修改 `pr.changed_files` 且禁止新增文件的约束下，不存在保留升级语义的最小修复点。
- 与本次情形相同的重复 PR #4819（同样升级 jetty 12.1.14）已因相同原因被关闭，进一步佐证该升级当前不可执行。

### 5. 结论与建议
本次为上游制品未发布的时序问题，无需修改代码。建议等待 `org.eclipse.jetty:jetty-home:12.1.14` 发布到 Maven 中央仓后重跑 CI，或按流程关闭/搁置该自动升级 PR。

## 潜在风险
无。本次未修改任何文件，不会引入新问题。
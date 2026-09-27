# CI 失败分析报告

## 基本信息
- PR: #4613 — 【自动升级】hudi容器镜像升级至1.2.1版本.
- 失败类型: dependency-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: Hudi多模块依赖缺失
- 新模式症状关键词: Could not resolve dependencies, Could not find artifact, hudi-flink2.1.x, hudi-flink-client, mvn clean package

## 根因分析

### 直接错误
```
#14 380.2 [ERROR] Failed to execute goal on project hudi-flink-client: Could not resolve dependencies for project org.apache.hudi:hudi-flink-client:jar:1.2.1
#14 380.2 [ERROR] dependency: org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 (compile)
#14 380.2 [ERROR] 	Could not find artifact org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 in Maven Central (https://repo.maven.apache.org/maven2)
#14 380.2 [ERROR] 	Could not find artifact org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 in confluent (https://packages.confluent.io/maven/)
#14 380.2 [ERROR] 	Could not find artifact org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 in jitpack.io (https://jitpack.io)
#14 380.2 [ERROR] 	Could not find artifact org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 in cloudera-repo-releases (https://repository.cloudera.com/artifactory/public/)
#14 ERROR: process "/bin/sh -c mvn clean package -DskipTests -Dspark3.5 -Dscala-2.12" did not complete successfully: exit code: 1
```
Reactor 摘要显示 `hudi-flink-client` 构建时前置模块（至 `hudi-cli`）全部 SUCCESS，`hudi-flink-client` FAILURE 之后所有 `hudi-flink*` 模块（`hudi-flink-datasource`、`hudi-flink`、`hudi-flink2.1-bundle`）均被 SKIPPED。

### 根因定位
- 失败位置: `Bigdata/hudi/1.2.1/24.03-lts-sp4/Dockerfile:21`（`RUN mvn clean package -DskipTests -Dspark3.5 -Dscala-2.12`）
- 失败原因: 该 Maven 命令对完整 reactor 全量构建，`hudi-flink-client` 模块引用了同项目内部制品 `org.apache.hudi:hudi-flink2.1.x:jar:1.2.1`，但该制品对应的 Flink 模块在 reactor 中排在 `hudi-flink-client` 之后（摘要中 `hudi-flink-datasource`/`hudi-flink` 位于其后且被 SKIPPED），导致构建顺序上该内部依赖尚未产出，Maven 退而从远程仓库解析并全部失败。

### 与 PR 变更的关联
直接相关。PR 新增的 Dockerfile 使用 `git clone --branch release-${VERSION}` 拉取 hudi 1.2.1 源码，并在 `Dockerfile:21` 执行全量 `mvn clean package -DskipTests -Dspark3.5 -Dscala-2.12`。日志中 `#14 > [builder 6/6] RUN mvn clean package...` 与 `Dockerfile:21` 的定位完全一致，说明失败由该 PR 新增的构建命令触发。PR 其余文件（README.md、doc/image-info.yml、meta.yml）为元数据/文档变更，与本次失败无关。

## 修复方向

### 方向 1（置信度: 中）
调整 Maven 构建范围，仅构建镜像实际需要的模块及其依赖（例如通过 `-pl` 指定 `packaging/hudi-cli-bundle`、`packaging/hudi-spark-bundle` 并用 `-am` 带上依赖），避免把会解析内部 Flink 制品的 `hudi-flink-client` 纳入构建。镜像运行阶段 `COPY` 的正是这两个 bundle 的产物，故该方向与 Dockerfile 需求一致。

### 方向 2（置信度: 中）
若需保留全量构建，则在命令中显式排除/跳过 Flink 相关模块（如 `-pl '!hudi-flink-client'` 或 Hudi 1.2.1 支持的 Flink 跳过参数/profile），使 reactor 不再尝试解析内部未产出的 `hudi-flink2.1.x` 制品。

### 方向 3（置信度: 低）
补充正确的 profile/版本参数以激活能产出 `hudi-flink2.1.x` 的模块，使 reactor 内部依赖闭环。该方向需先确认 hudi 1.2.1 中 `hudi-flink2.1.x` 制品由哪个模块、在何种 flag 下产出。

## 需要进一步确认的点
- 日志未提供 hudi 1.2.1 的 `pom.xml`/module 列表，无法确认产出 `org.apache.hudi:hudi-flink2.1.x:jar:1.2.1` 的确切模块名与激活条件。需查阅 hudi `release-1.2.1` 源码确认。
- 无法确认 hudi 1.2.1 是否支持 `-pl`/`-am` 仅构建 cli-bundle 与 spark-bundle 的路径（相关打包模块目录名需以上游实际为准）。
- 无法确认失败是否仅在 x86-64 与 aarch64 之一发生；本日志仅含单一 builder 阶段输出，另一架构 job 的日志未提供。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
本次失败不涉及正则 patch 外部源文件，但若修复方向选择修改 Maven 构建命令（`-pl`/`-pl '!...'`/profile 参数），仍需验证：
- code-fixer 必须从 apache/hudi 的 `release-1.2.1` 分支获取根 `pom.xml` 与 `packaging/` 下各模块 `pom.xml`，确认目标构建模块的准确 artifactId 与目录名，以及 `hudi-flink2.1.x` 制品的产出模块，验证所选 `-pl`/排除参数确实能形成依赖闭环后再提交。

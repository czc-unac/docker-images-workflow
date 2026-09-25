# CI 失败分析报告

## 基本信息
- PR: #4447 — 【自动升级】hudi容器镜像升级至1.2.1版本.
- 失败类型: dependency-error
- 置信度: 中
- 知识库匹配: 新模式
- 新模式标题: 依赖模块未构建
- 新模式症状关键词: Could not find artifact, Maven Central, hudi-flink2.1.x, DependencyResolutionException, mvn clean package

## 根因分析

### 直接错误
```
#13 699.0 [ERROR] Failed to execute goal on project hudi-flink-client: Could not resolve dependencies for project org.apache.hudi:hudi-flink-client:jar:1.2.1
#13 699.0 [ERROR] dependency: org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 (compile)
#13 699.0 [ERROR] 	Could not find artifact org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 in Maven Central (https://repo.maven.apache.org/maven2)
#13 699.0 [ERROR] 	Could not find artifact org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 in confluent (https://packages.confluent.io/maven/)
#13 699.0 [ERROR] 	Could not find artifact org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 in jitpack.io (https://jitpack.io)
#13 699.0 [ERROR] 	Could not find artifact org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 in cloudera-repo-releases (https://repository.cloudera.com/artifactory/public/)
#13 699.0 [ERROR] 	org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 was not found in ... during a previous attempt...
#13 699.0 [INFO] ------------------------------------------------------------------------
#13 699.0 [INFO] hudi-flink-client .................................. FAILURE [ 14.980 s]
#13 699.0 [INFO] hudi-spark3.5-bundle_2.12 .......................... SKIPPED
#13 699.0 [INFO] hudi-cli-bundle_2.12 ............................... SKIPPED
#13 699.0 [INFO] BUILD FAILURE
#13 ERROR: process "/bin/sh -c mvn clean package -DskipTests -Dspark3.5 -Dscala-2.12" did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Bigdata/hudi/1.2.1/24.03-lts-sp4/Dockerfile:21`（`RUN mvn clean package -DskipTests -Dspark3.5 -Dscala-2.12`）
- 失败原因: 该命令对 Hudi 1.2.1 全量 reactor 构建时，`hudi-flink-client` 模块被纳入构建，但它依赖的本地模块 `org.apache.hudi:hudi-flink2.1.x:jar:1.2.1` 未出现在当前 reactor 中（Flink 2.1 相关 profile 未被激活），Maven 遂转向远程仓库解析该 `1.2.1` 版本制品，因该版本仅存在于本次源码树、尚未发布到任何远程仓库，四个仓库全部报 "Could not find artifact"，依赖解析失败。

### 与 PR 变更的关联
直接相关。本 PR 新增 `Bigdata/hudi/1.2.1/24.03-lts-sp4/Dockerfile`，其构建命令 `mvn clean package -DskipTests -Dspark3.5 -Dscala-2.12` 需要 Hudi 1.2.1 源码树的全量 reactor 能自洽。日志中 `-Dspark3.5 -Dscala-2.12` 仅激活了 Spark 侧 profile，导致 Flink 2.1 模块未参与构建，从而触发 `hudi-flink-client` 的依赖解析失败。日志末尾为 `BUILD FAILURE` / `Finished: FAILURE`，不存在"日志成功但状态失败"的情形。

## 修复方向

### 方向 1（置信度: 中）
仅构建本镜像实际需要的模块，避免纳入会失败或不需要的 Flink 模块。Dockerfile 真正 COPY 的产物只有：
- `packaging/hudi-cli-bundle/target/hudi-cli-bundle_2.12-${VERSION}.jar`
- `packaging/hudi-spark3.5-bundle/target/hudi-spark3.5-bundle_2.12-${VERSION}.jar`

因此可将 `mvn` 命令限定为上述两个 bundling 模块及它们所需的上游模块（如 `-pl packaging/hudi-cli-bundle,packaging/hudi-spark3.5-bundle -am`），或显式排除 `hudi-flink-client`/Flink 相关模块，使 reactor 不再包含依赖未构建模块的 `hudi-flink-client`。

### 方向 2（可选，置信度: 中）
在保持全量 reactor 思路的前提下，补充激活 Hudi 1.2.1 中对应的 Flink 2.1 profile（使 `hudi-flink2.1.x` 模块进入 reactor 并被本地构建），从而让 `hudi-flink-client` 的依赖在本地即可解析，无需远程仓库。

> 说明：方向 1 更贴合"只产出 spark/cli bundle"的镜像意图，且不依赖上游 profile 命名细节，推荐优先验证。

## 需要进一步确认的点
1. 需要对照 `Bigdata/hudi/1.2.0/24.03-lts-sp4/Dockerfile` 中 1.2.0 版本所使用的 `mvn` 命令，确认 1.2.0 为何未出现同一问题（是否本来就使用了模块裁剪 `-pl`，或 Hudi 1.2.0 无 Flink 2.1 模块）。
2. 需要从 Hudi 1.2.1 上游（`release-1.2.1` 标签）确认实际可用的 profile/属性名（例如 Flink 2.1 对应 profile 是 `flink2.1` 还是其他），以及 `hudi-flink2.1.x` 模块所在目录，才能保证方向 2 的正则/参数正确。
3. 需确认 `packaging/hudi-spark3.5-bundle` 与 `packaging/hudi-cli-bundle` 的确切模块坐标与相对路径，以确定方向 1 的 `-pl` 参数是否准确。

## 修复验证要求
本失败不涉及"修改正则 patch 第三方源文件"，无需上游文件正则匹配验证。但若采用**方向 1**，code-fixer 必须先确认 Hudi 1.2.1 `pom.xml` 中 bundling 模块的准确模块坐标（artifactId）与目录路径；若采用**方向 2**，必须先从 apache/hudi `release-1.2.1` 获取其根 `pom.xml`，核实触发 `hudi-flink2.1.x` 模块构建所需的 profile 名称，确认后再提交，不可假设 profile 名一定为 `flink2.1`。

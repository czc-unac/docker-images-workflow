# CI 失败分析报告

## 基本信息
- PR: #4711 — 【自动升级】gluten容器镜像升级至1.7.0版本.
- 失败类型: dependency-error
- 置信度: 中
- 知识库匹配: 新模式
- 新模式标题: Maven依赖制品缺失
- 新模式症状关键词: Could not resolve dependencies, org.apache.spark, spark-sql_2.12:jar:4.0.2, gcs-maven-central-mirror, DependencyResolutionException, BUILD FAILURE

> 前置一致性检查：`ci.logs` 末尾为 `Build step 'Execute shell' marked build as failure` / `Finished: FAILURE`，非成功日志，故进入正常分析流程（不适用"证据不足"终止条件）。

## 根因分析

### 直接错误
```
#12 206.1 [ERROR] Failed to execute goal on project gluten-core: Could not resolve dependencies for project org.apache.gluten:gluten-core:jar:1.7.0:
The following artifacts could not be resolved: org.apache.spark:spark-sql_2.12:jar:4.0.2, org.apache.spark:spark-core_2.12:jar:4.0.2,
org.apache.spark:spark-catalyst_2.12:jar:4.0.2, org.apache.spark:spark-hive_2.12:jar:4.0.2, org.apache.spark:spark-kvstore_2.12:jar:4.0.2,
org.apache.spark:spark-network-common_2.12:jar:4.0.2, org.apache.spark:spark-network-shuffle_2.12:jar:4.0.2,
org.apache.spark:spark-core_2.12:jar:tests:4.0.2, org.apache.spark:spark-sql_2.12:jar:tests:4.0.2, org.apache.spark:spark-catalyst_2.12:jar:tests:4.0.2:
Failure to find org.apache.spark:spark-sql_2.12:jar:4.0.2 in https://maven-central.storage-download.googleapis.com/maven2/ was cached in the local repository,
resolution will not be reattempted until the update interval of gcs-maven-central-mirror has elapsed or updates are forced -> [Help 1]
...
#12 206.1 [INFO] Gluten Parent Pom .................................. SUCCESS [01:22 min]
#12 206.1 [INFO] Gluten Core ........................................ FAILURE [01:29 min]
#12 206.1 [INFO] Gluten Package ..................................... FAILURE [ 20.224 s]
#12 206.1 [INFO] BUILD FAILURE
#12 ERROR: process "/bin/sh -c if [ -x ./build/mvn ]; then ... mvn -DskipTests -T1C package; ... fi" did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Bigdata/gluten/1.7.0/24.03-lts-sp4/Dockerfile:23-27`（`RUN if [ -x ./build/mvn ] ... mvn -DskipTests -T1C package` 步骤）
- 失败模块: Maven reactor 中的 `gluten-core`、`gluten-package`（`org.apache.gluten:*:jar:1.7.0`）
- 失败原因: gluten 1.7.0 构建时无法从配置的 `gcs-maven-central-mirror`（`https://maven-central.storage-download.googleapis.com/maven2/`）解析出 `org.apache.spark:*_2.12:4.0.2` 系列制品（spark-sql / spark-core / spark-catalyst / spark-hive / spark-kvstore / spark-network-common / spark-network-shuffle 及其 tests 分类器），Maven 报 `DependencyResolutionException`，且该失败结果被缓存后不再重试。

### 关键佐证（区分"网络问题"与"制品缺失"）
- 同一构建中大量其他依赖（`junit-4.13.1.jar`、`hamcrest-core-1.3.jar`、`protobuf-java-3.23.4.jar`、`byte-buddy-1.9.3.jar`、`objenesis-2.6.jar`、`scalatestplus-*` 等）均从 `gcs-maven-central-mirror` **成功下载**，说明到该镜像站的网络连通正常，失败是**制品粒度**的问题，而非通用网络/超时问题。
- 报错明确为 `Could not find artifact` / `Failure to find ... was cached`，即镜像中查无此 `4.0.2` 版本制品，而非连接超时或 5xx。

### 与 PR 变更的关联
- 本 PR 为"自动升级"新增 `Bigdata/gluten/1.7.0/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=1.7.0`、`git clone --depth 1 --branch v${VERSION}` 克隆 gluten `v1.7.0`，随后执行 `mvn -DskipTests -T1C package`（未指定 Spark profile / spark.version）。
- 失败正是发生在该新增 Dockerfile 的首次构建中，gluten 1.7.0 的默认构建配置解析 Spark 4.0.2 制品失败，**与本次 PR 直接相关**。
- 同一 PR 中 `README.md`、`doc/image-info.yml`、`meta.yml` 的改动为元数据/文档，与本次 Maven 解析失败无因果关系。

## 修复方向

### 方向 1（置信度: 中）
在构建 gluten 前显式指定一个在当前 Maven 镜像/仓库中确实可解析的 Spark 版本或 profile（gluten 通过 `pom.xml` 的 Spark profile / `spark.version` 属性选择目标 Spark 版本）。需先确认 gluten 1.7.0 支持的 Spark profile 集合及对应版本是否在镜像中可用。

### 方向 2（置信度: 中）
若确认 Spark 4.0.2 制品在 Maven Central 存在、仅 `gcs-maven-central-mirror` 缺失或同步滞后，则调整 Maven 仓库/镜像配置指向包含该制品的仓库；并清理/强制更新本地仓库缓存（`-U`），以突破日志中"resolution will not be reattempted ... was cached"的限制。若制品本身不存在，则本方向无效。

### 方向 3（置信度: 低）
若为自动升级脚本选取的版本组合本身不自洽（gluten 1.7.0 与 Spark 4.0.2 不匹配），则需回退/修正自动升级产生的 VERSION 与目标依赖版本组合。

## 需要进一步确认的点
1. `org.apache.spark:spark-sql_2.12:4.0.2`（及同系列制品）在 Maven Central 是否真实存在——直接访问 `https://repo.maven.apache.org/maven2/org/apache/spark/spark-sql_2.12/4.0.2/` 核实。
2. gluten `v1.7.0` 源码 `pom.xml` 中默认 `spark.version`、可用的 Spark profile（如 `-Pspark-3.5` / `-Pspark-4.0` 等）及默认激活的 profile，确认 4.0.2 的来源。
3. gluten 1.7.0 官方发布说明/兼容矩阵中声明的目标 Spark 版本。
4. CI 构建环境中 gluten `build/mvn` 使用的 Maven settings/mirror 配置（为何只走 `gcs-maven-central-mirror`）。
5. 日志中 `Downloading from central: https://repo.maven.apache.org/maven2/.../4.0.2/...` 的下载是否最终成功；日志被截断，无法确认 central 侧结果。

## 修复验证要求
> 本修复方向不涉及对第三方/上游源文件的"正则 patch"，但仍需 code-fixer 在提交前完成以下验证（置信度为"中"，不可假设方向 1/2 一定正确）：

1. 先核实根因：确认 Spark 4.0.2 制品是否存在于 Maven Central；若不存在，方向 2 无效，只采用方向 1。
2. 从 `apache/gluten` 的 `v1.7.0` 标签拉取 `pom.xml`（及 `build/mvn`、相关 `maven.config`/settings 文件），确认其默认 Spark 版本与可用 profile。
3. 以 `spark.version`/profile 的候选值逐一验证对应 `org.apache.spark:*_2.12:<ver>` 制品在目标镜像/仓库可解析，再据此修改 Dockerfile 的构建命令。
4. 不得在未确认制品可用性的前提下直接改写依赖版本；若无法确认，应保持方向标注为"证据不足"并回报，而非盲目提交。

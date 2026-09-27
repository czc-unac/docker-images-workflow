# CI 失败分析报告

## 基本信息
- PR: #4604 — 【自动升级】gluten容器镜像升级至1.7.0版本.
- 失败类型: `dependency-error`
- 置信度: 中
- 知识库匹配: 新模式
- 新模式标题: Maven依赖解析失败
- 新模式症状关键词: Could not resolve dependencies, org.apache.spark, spark-sql_2.12:4.0.2, gcs-maven-central-mirror, was cached, resolution will not be reattempted

## 根因分析

### 直接错误
```
#12 193.9 [ERROR] Failed to execute goal on project gluten-package: Could not resolve dependencies for project org.apache.gluten:gluten-package:jar:1.7.0: The following artifacts could not be resolved: org.apache.spark:spark-sql_2.12:jar:4.0.2, org.apache.spark:spark-core_2.12:jar:4.0.2, org.apache.spark:spark-catalyst_2.12:jar:4.0.2, org.apache.spark:spark-hive_2.12:jar:4.0.2: Could not find artifact org.apache.spark:spark-sql_2.12:jar:4.0.2 in gcs-maven-central-mirror (https://maven-central.storage-download.googleapis.com/maven2/) -> [Help 1]
#12 193.9 [ERROR] Failed to execute goal on project gluten-core: Could not resolve dependencies ... Failure to find org.apache.spark:spark-sql_2.12:jar:4.0.2 in https://maven-central.storage-download.googleapis.com/maven2/ was cached in the local repository, resolution will not be reattempted until the update interval of gcs-maven-central-mirror has elapsed or updates are forced -> [Help 1]
#12 193.9 [INFO] Gluten Core ........................................ FAILURE [01:30 min]
#12 193.9 [INFO] Gluten Package ..................................... FAILURE [ 25.355 s]
#12 193.9 [INFO] BUILD FAILURE
#12 ERROR: process "/bin/sh -c if [ -x ./build/mvn ]; then ./build/mvn -DskipTests -T1C package; ..." did not complete successfully: exit code: 1
```
末尾为 `Finished: FAILURE`，日志与实际失败状态一致（前置检查通过）。

### 根因定位
- 失败位置: `Bigdata/gluten/1.7.0/24.03-lts-sp4/Dockerfile:23`（`RUN ... mvn -DskipTests -T1C package` 步骤）
- 失败原因: Gluten 1.7.0 构建需要 `org.apache.spark:*_2.12:4.0.2` 一组制品，但配置的 Maven 仓库 `gcs-maven-central-mirror`（`https://maven-central.storage-download.googleapis.com/maven2/`）中不存在这些制品；Maven 将该"找不到"结果缓存进本地仓库（`resolution will not be reattempted ... or updates are forced`），导致后续即使有 `central` 可用也不再重试，最终 `gluten-core`、`gluten-package` 模块依赖解析失败、构建中断。

### 与 PR 变更的关联
PR 新增了 `Bigdata/gluten/1.7.0/24.03-lts-sp4/Dockerfile`，该 Dockerfile 首次引入 gluten 1.7.0 的 Maven 构建。失败 100% 发生在此新增文件的构建步骤内，属于本 PR 直接触发。其余改动（README.md、doc/image-info.yml、meta.yml）为元数据登记，不参与该构建过程。

## 修复方向

### 方向 1（置信度: 中）
调整 Maven 依赖解析来源：让 Maven 从确实包含 Spark 4.0.2 制品的仓库（如 Maven Central `repo.maven.apache.org`）解析，而不是在 `gcs-maven-central-mirror` 上失败并缓存负结果。可选手段包括：在构建时强制刷新（`-U`）以避免命中本地缓存的失败解析、清理/更换本地仓库、或修正 gluten 构建使用的仓库/镜像配置（`./build/mvn` 或其 settings/pom 中的 repository、mirror 定义），使 Spark 4.0.2 可从有效源获取。

### 方向 2（可选）
核对 gluten 1.7.0 实际要求的 Spark 版本：确认 Spark 4.0.2 是否已在公共仓库发布并被镜像覆盖。若该 Spark 版本本身尚不存在/未同步，则应选择与 gluten 1.7.0 兼容且仓库中确实存在的 Spark 版本，或调整构建所用的 profile/Scala 组合。日志中同时出现 `Downloading from central: https://repo.maven.apache.org/maven2/org/apache/spark/...4.0.2/...`，说明构建既有 gcs 镜像源也有 central 源，需先厘清两源各自是否真正提供 4.0.2。

## 需要进一步确认的点
- gluten 1.7.0 的 pom（或 `build/mvn`、`.mvn/` 配置）中 `spark.version` 的实际取值，以及与 `4.0.2` 对应的 Scala 版本（日志为 `_2.12`）。
- `gcs-maven-central-mirror` 与 `central` 两个仓库在 gluten 构建配置中的声明位置、优先级与 mirror 覆盖关系，为何 spark 制品只在 gcs 源上尝试并缓存失败。
- `org.apache.spark:spark-sql_2.12:4.0.2` 是否已存在于 Maven Central；若不存在，需确认 gluten 1.7.0 期望的正确 Spark 版本。
- 该失败是否同时出现在 aarch64 架构构建 job（日志仅显示单次 Docker build 失败，未显示架构维度）。

## 修复验证要求
本失败不涉及正则 patch 上游源文件，故不适用 getdeps/fetcher 类正则验证。但因置信度为"中"，code-fixer 在提交前必须完成以下验证：
1. 拉取 gluten `v1.7.0`（以 Dockerfile `ARG VERSION=1.7.0` 为准）的构建配置，确认其声明的 Spark 版本与仓库来源，验证 `spark-sql_2.12:4.0.2` 所需源确实提供该制品。
2. 本地或临时环境复现 `mvn -DskipTests -T1C package` 所走的仓库解析路径，确认修改后 Maven 能真正下载到 `org.apache.spark:*_2.12:4.0.2` 而不再命中缓存的失败解析。
3. 若改动涉及仓库/mirror 配置，需确认改动后不会破坏其它已成功下载的依赖（junit、protobuf 等当前经 gcs 镜像下载成功）。

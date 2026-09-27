# 修复摘要

## 修复的问题
gluten 1.7.0 镜像构建时 Maven 解析 `org.apache.spark:*_2.12:4.0.2` 失败，原因是构建未指定 Spark profile，导致上游 pom 默认的 Scala 2.12 与 Spark 4.0.2 组合不兼容/制品不存在。通过显式指定 `-Pspark-3.5` 修复依赖解析。

## 修改的文件
- `Bigdata/gluten/1.7.0/24.03-lts-sp4/Dockerfile`: 在 Maven 构建命令中新增 `-Pspark-3.5` profile（`./build/mvn` 与 `mvn` 两个分支均添加）。

## 修复逻辑

### 根因（与 CI 分析报告的差异说明）
CI 分析报告推测根因是 Maven 仓库/mirror（`gcs-maven-central-mirror`）问题。实际验证后发现真正根因是**上游 pom 的默认属性组合不一致**：

- gluten `v1.7.0` 的 `pom.xml` 默认属性为 `spark.version=4.0.2`，但 `scala.binary.version=2.12`、`sparkshim.artifactId=spark-sql-columnar-shims-spark35`。
- 经实测：`org.apache.spark:spark-sql_2.12:4.0.2` 在 Maven Central（repo1.maven.org、repo.maven.apache.org）和 `gcs-maven-central-mirror` 上**均返回 404**，即 Spark 4.x 只发布 Scala 2.13 制品（`spark-sql_2.13:4.0.2` 返回 200），不存在 2.12 制品。因此更换/刷新 Maven 仓库无法解决问题。
- Dockerfile 第 23 行直接执行 `./build/mvn -DskipTests -T1C package`，未指定任何 Spark/Scala profile，故使用了上述不一致的默认值，最终报 `Could not resolve dependencies ... spark-sql_2.12:4.0.2`。
- 对比：1.6.0 镜像的 Dockerfile 使用完全相同的命令却能构建成功，因为 `v1.6.0` 的 pom 默认值为 `spark.version=3.5.5`（存在 `_2.12` 制品）。本失败是 1.7.0 上游默认值变更后本仓库 Dockerfile 未同步指定 profile 所致。

### 修复方式
为构建命令显式加上 `-Pspark-3.5`，使 `spark.version` 被覆盖为 3.5.5，与默认 Scala 2.12、`sparkshim=spark35`、`sparkbundle.version=3.5` 完全一致，恢复与 1.6.0 镜像相同的、已验证可用的依赖组合。

### 验证结果
1. 已从上游 `apache/gluten` tag `v1.7.0` 拉取 `pom.xml`/`build/mvn`/`dev/release/build-release.sh` 核对：
   - `pom.xml` 顶部确实为 `spark.version=4.0.2` + `scala.binary.version=2.12`（互相矛盾）。
   - `spark-3.5` profile 将 `spark.version` 覆盖为 `3.5.5`，并设置 `sparkbundle.version=3.5`、`sparkshim.artifactId=spark-sql-columnar-shims-spark35`，与默认其余属性一致。
   - `dev/release/build-release.sh` 对 Spark 3.5 使用 `-Pjava-17 -Pbackends-velox -Pspark-3.5`（Java 17 由镜像中的 `java-17-openjdk-devel` 经 JDK 自动激活，`-Pspark-3.5` 即所需最小 profile）。
2. 制品可用性验证（HTTP 状态码）：
   - `spark-sql_2.12:4.0.2`：central / gcs mirror 均 404。
   - `spark-sql_2.12:3.5.5`、`spark-core_2.12:3.5.5`、`spark-catalyst_2.12:3.5.5`、`spark-hive_2.12:3.5.5`：central 与 `gcs-maven-central-mirror` 均 200。
3. 仓库既有先例：`Bigdata/celeborn/0.6.3/*/Dockerfile` 同样显式使用 `-Pspark-3.5`，本修复与该约定一致。
4. `scala-2.12` 是 `activeByDefault` 唯一默认 profile，命令行激活 `spark-3.5` 即使使其失效，pom 顶层默认属性仍为 Scala 2.12.18，结果不变，无副作用。

说明：本修复不涉及对上游源文件的正则 patch，故不适用 getdeps/fetcher 类正则验证。

## 潜在风险
- 该修复将镜像内的 Spark 版本固定为 3.5.5（与 1.6.0 镜像一致）。若期望 gluten 1.7.0 镜像提供 Spark 4.0 支持，则应改为 `-Pspark-4.0 -Pscala-2.13`（Java 17 自动激活）；但该组合会切换到 Scala 2.13 编译，改动面与风险更大，且与现有镜像行为不一致，故未采用。
- 仅修改了构建 profile，未改动 Maven 仓库/mirror 配置，其他依赖（junit、protobuf 等）的下载路径不受影响。
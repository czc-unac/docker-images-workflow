# 修复摘要

## 修复的问题
gluten 1.7.0 在未指定 Spark profile 时，默认按 Spark 4.0.2 + Scala 2.12 解析依赖，而该制品组合在 Maven 仓库中不存在，导致 `mvn package` 依赖解析失败；通过显式启用上游默认的 `spark-3.5` profile 修复。

## 修改的文件
- `Bigdata/gluten/1.7.0/24.03-lts-sp4/Dockerfile`: 在第 23-27 行的 Maven 构建命令中为 `./build/mvn` 与 `mvn` 两个分支均增加 `-Pspark-3.5` 参数。

## 修复逻辑
- 分析报告根因定位正确：失败发生在新增 Dockerfile 的 `mvn -DskipTests -T1C package` 步骤，`gluten-core` 无法解析 `org.apache.spark:spark-sql_2.12:jar:4.0.2` 等制品。
- 核实根因（已从上游 `apache/gluten` 的 `v1.7.0` 标签拉取 `pom.xml`、`docs/get-started/build-guide.md` 验证）：
  - `pom.xml` 默认属性为 `<spark.version>4.0.2</spark.version>` + `<scala.binary.version>2.12</scala.binary.version>`；但 Spark 4.0 只发布 Scala 2.13 制品。实际访问 Maven Central 核实：`org/apache/spark/spark-sql_2.12/4.0.2/` 返回 **404**，而 `spark-sql_2.13/4.0.2/` 返回 **200**。因此这不是镜像同步/网络问题，而是默认（无 profile）构建的 Spark/Scala 版本组合本身不存在。
  - `spark-4.0` profile 明确要求 `-Pjava-17 -Pscala-2.13` 同时启用（构建指南与 `dev/release/build-release.sh` 均如此），说明 1.7.0 不能依赖无 profile 的默认组合。
  - 上游 `docs/get-started/build-guide.md` 的 Maven 参数表将 `spark-3.5` 标注为默认启用（`enabled`），其余 Spark profile 均为 `disabled`；官方 `build_bundle_package.yml` 的默认输入同样为 `spark-3.5`。且前一版本 `1.6.0` 的 Dockerfile 使用相同命令、其默认 `spark.version=3.5.5`，镜像即为 Spark 3.5.5 制品。
- 因此采用最小化修复：显式 `-Pspark-3.5`，使构建解析到 `org.apache.spark:*_2.12:3.5.5`。已逐项核实 Maven Central 存在对应制品（jar 与 tests 分类器）：`spark-sql/core/catalyst/hive/kvstore/network-common/network-shuffle/tags_2.12:3.5.5` 均返回 200，`spark-core/sql/catalyst_2.12:3.5.5-tests` 亦返回 200。
- 该改动仅触及 `pr.changed_files` 中的 Dockerfile，未修改上游源码，不涉及正则 patch。

## 潜在风险
- 该修复将镜像的构建目标固定为 Spark 3.5.5（与 1.6.0 镜像行为一致）。若后续希望提供 Spark 4.0/4.1 版本制品，需另建镜像并显式使用 `-Pjava-17 -Pscala-2.13 -Pspark-4.0`（或 `spark-4.1`）。
- 若上游在后续补丁版本中修正默认属性，本参数仍可安全保留，无副作用。
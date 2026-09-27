# 修复摘要

## 修复的问题
修复 Hudi 1.2.1 镜像构建时因 `-Dspark3.5` 激活 spark3.5 profile 而自动停用默认 flink2.1 profile，导致 `hudi-flink-client` 依赖的内部制品 `hudi-flink2.1.x` 不在 reactor 中、Maven 转向远程仓库解析失败的问题。

## 修改的文件
- `Bigdata/hudi/1.2.1/24.03-lts-sp4/Dockerfile`: 第 21 行 Maven 命令由全量构建改为仅构建镜像实际需要的两个 bundle 及其依赖：
  - 原：`mvn clean package -DskipTests -Dspark3.5 -Dscala-2.12`
  - 新：`mvn clean package -DskipTests -Dspark3.5 -Dscala-2.12 -pl packaging/hudi-cli-bundle,packaging/hudi-spark-bundle -am`

## 修复逻辑
根因来自上游 apache/hudi `release-1.2.1` 的 POM 结构（已通过 WebFetch 拉取实际源文件验证）：

1. 根 `pom.xml` 中 `hudi-client` 声明在 `hudi-flink-datasource` 之前；`hudi-client` 的子模块 `hudi-flink-client`（`hudi-client/hudi-flink-client/pom.xml:65`）硬依赖 `${hudi.flink.module}`，而根 POM 默认值为 `hudi-flink2.1.x`。
2. 产出 `org.apache.hudi:hudi-flink2.1.x:1.2.1` 的模块是 `hudi-flink-datasource/hudi-flink2.1.x`，它仅由 `flink2.1` profile（`<activeByDefault>true</activeByDefault>`）加入 reactor。
3. Dockerfile 传入 `-Dspark3.5`，通过 property activation 激活 `spark3.5` profile；按 Maven 规则，同 POM 内任一 profile 被 activation config 激活时，所有 `activeByDefault` 的 profile 会被停用。因此 `flink2.1` profile 被停用，`hudi-flink2.1.x` 不在 reactor 内，`hudi-flink-client` 只能从远程仓库解析该内部制品并失败（与 CI 日志完全吻合）。
4. 采用报告"方向 1"：用 `-pl packaging/hudi-cli-bundle,packaging/hudi-spark-bundle -am` 仅构建 Dockerfile 运行阶段真正 COPY 的两个 bundle 及其依赖闭包。已验证 `hudi-cli-bundle` 依赖 `hudi-cli`、`hudi-utilities_2.12`，`hudi-spark-bundle` 依赖 `hudi-spark_2.12`、`hudi-hadoop-mr`、`hudi-hive-sync`、`hudi-timeline-service`；这些模块均无任何 flink 引用，因此 reactor 不再包含 `hudi-flink-client`，从根本上避免解析未产出的 `hudi-flink2.1.x`。
5. 目标产物名与 COPY 行一致：`hudi-cli-bundle_2.12-1.2.1.jar`、`hudi-spark3.5-bundle_2.12-1.2.1.jar`。

其他验证：该修复与仓库历史中针对同一 Dockerfile 的既有 CI 修复提交 `55a459f32`（PR #4447）完全一致，说明这是已被验证过的正确改法。

## 潜在风险
无。仅缩小 Maven 构建范围到镜像实际使用的两个 bundle，未改变运行阶段 COPY 路径与产物名；`README.md`、`doc/image-info.yml`、`meta.yml` 为文档/元数据，无需改动。
# 修复摘要

## 修复的问题
修正 Hudi 1.2.1 镜像 Dockerfile 的 Maven 构建范围，避免构建未纳入 Reactor 的 `hudi-flink2.1.x` 内部依赖导致 `hudi-flink-client` 依赖解析失败（BUILD FAILURE）。

## 修改的文件
- `Bigdata/hudi/1.2.1/24.03-lts-sp4/Dockerfile`: 第 21 行在原 `mvn clean package` 命令后追加 `-pl packaging/hudi-cli-bundle,packaging/hudi-spark-bundle -am`，将整仓 Reactor 构建限定为本镜像实际需要的两个打包模块及其上游依赖。

## 修复逻辑
分析报告指出 `hudi-flink-client` 模块依赖 `org.apache.hudi:hudi-flink2.1.x:jar:1.2.1`，而该模块未出现在 Reactor 中，Maven 转而向远程仓库解析并失败。

经核对上游 `apache/hudi` 仓库 `release-1.2.1` 源码确认根因：
- 根 `pom.xml` 中 `hudi-client` 聚合模块（第 40 行）无条件包含子模块 `hudi-client/hudi-flink-client`；
- `hudi-flink-client/pom.xml` 依赖 `${hudi.flink.module}`，其默认值为 `hudi-flink2.1.x`（根 pom 第 170 行）；
- 模块 `hudi-flink2.1.x` 仅在 `flink2.1` profile 中声明（根 pom 第 2961-2974 行），该 profile 标记为 `activeByDefault`；
- 传入 `-Dspark3.5` 会激活 `spark3.5` profile，按照 Maven 规则同一 POM 内 `activeByDefault` profile 会被自动停用，导致 `flink2.1` 被停用、`hudi-flink2.1.x` 未进入 Reactor，而 `hudi-flink-client` 仍被构建，最终解析失败。

修复采用分析报告的方向 1（限定构建模块），仅构建最终 COPY 的两个 bundle 及必要上游：`hudi-cli-bundle` 与 `hudi-spark3.5-bundle`。经核对与验证，`-am` 带入的上游模块全部为 spark/cli/utilities 链路，均不依赖任何 flink 模块。

### 验证过程与结果
1. 通过 `git clone --filter=blob:none --branch release-1.2.1` 实际获取上游源码，比对根 pom、`hudi-client/pom.xml`、`hudi-flink-client/pom.xml`、`hudi-spark-bundle/pom.xml`、`hudi-cli-bundle/pom.xml` 等确认真实模块/依赖结构。
2. 在本环境安装的 Maven（Java 17，仅为校验 Reactor 组合）上执行：
   - 修复前：`mvn validate -DskipTests -Dspark3.5 -Dscala-2.12`，Reactor 中出现 `hudi-flink-client`，但**不存在** `hudi-flink2.1.x`，复现根因。
   - 修复后：`mvn validate -DskipTests -Dspark3.5 -Dscala-2.12 -pl packaging/hudi-cli-bundle,packaging/hudi-spark-bundle -am`，退出码 0；Reactor Build Order 为 25 个模块，**不含任何 flink 模块**，且包含 `hudi-spark3.5-bundle_2.12` 与 `hudi-cli-bundle_2.12`。
3. 产物路径与第 43-44 行 COPY 目标一致：`packaging/hudi-cli-bundle/target/hudi-cli-bundle_2.12-1.2.1.jar`、`packaging/hudi-spark-bundle/target/hudi-spark3.5-bundle_2.12-1.2.1.jar`（artifactId 分别为 `hudi-cli-bundle_${scala.binary.version}`、`hudi-spark${sparkbundle.version}-bundle_${scala.binary.version}`，在 `-Dspark3.5 -Dscala-2.12` 下即为该名称）。

说明：完整 `mvn package` 编译需下载 Hudi 全量依赖并以 Java 11 执行，耗时远超本环境限制，故验证止于 Reactor 组合与模块依赖关系层面；该层面的验证已直接证明 `hudi-flink-client` / `hudi-flink2.1.x` 不再进入构建。

## 潜在风险
- 构建范围由整仓收窄为两个 bundle 及其上游，不再构建 `hudi-flink`、`hudi-kafka-connect` 等无关模块。经核对以下游 bundle 的依赖闭包，运行期所需产物均仍被构建，第 43-46 行 COPY 的 jar、脚本、conf 均不受影响，风险低。
- 若未来该镜像需要额外的 Hudi 模块产物，需同步在 `-pl` 列表中追加对应模块。
- 未改动 README.md、doc/image-info.yml、meta.yml（分析报告已确认其为文档/元数据，非失败原因），符合最小化改动原则。
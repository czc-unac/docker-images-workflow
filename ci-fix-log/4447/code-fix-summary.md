# 修复摘要

## 修复的问题
Hudi 1.2.1 镜像构建时全量 reactor 构建 `hudi-flink-client` 因依赖未参与构建的 `org.apache.hudi:hudi-flink2.1.x:jar:1.2.1` 而依赖解析失败，现将 Maven 构建裁剪为仅构建镜像实际需要的两个 bundling 模块及其上游依赖。

## 修改的文件
- `Bigdata/hudi/1.2.1/24.03-lts-sp4/Dockerfile`: 第 21 行构建命令由 `mvn clean package -DskipTests -Dspark3.5 -Dscala-2.12` 改为 `mvn clean package -DskipTests -Dspark3.5 -Dscala-2.12 -pl packaging/hudi-cli-bundle,packaging/hudi-spark-bundle -am`。

## 修复逻辑
- 根因：分析报告中 `hudi-flink-client` 解析 `hudi-flink2.1.x:1.2.1` 失败，该版本仅存在于当前源码树、未发布到远程仓库；全量 reactor 在 `-Dspark3.5 -Dscala-2.12` 下未激活 Flink 2.1 模块，导致依赖解析转向远程仓库并全面失败。
- 采用分析报告的"方向 1"：Dockerfile 真正 COPY 的产物只有 `hudi-cli-bundle` 与 `hudi-spark3.5-bundle`，因此用 `-pl` 限定这两个模块、并用 `-am` 一并构建其上游依赖模块，从 reactor 中彻底排除 Flink 相关模块（含 `hudi-flink-client`）。
- 上游验证（已从 apache/hudi `release-1.2.1` 获取根 `pom.xml`、`packaging/hudi-spark-bundle/pom.xml`、`packaging/hudi-cli-bundle/pom.xml`、`hudi-flink-datasource/pom.xml`、`hudi-flink-datasource/hudi-flink/pom.xml`、`hudi-utilities/pom.xml`）：
  - 根 `pom.xml` 的 `<modules>` 明确列出 `packaging/hudi-spark-bundle` 与 `packaging/hudi-cli-bundle`，故 `-pl` 的目录路径准确有效。
  - `packaging/hudi-spark-bundle` 的 artifactId 为 `hudi-spark${sparkbundle.version}-bundle_${scala.binary.version}`，在 `-Dspark3.5 -Dscala-2.12` 下即 `hudi-spark3.5-bundle_2.12`，其 shade `finalName` 为 `${artifactId}-${version}`，与 Dockerfile 第 44 行 COPY 的 `hudi-spark3.5-bundle_2.12-1.2.1.jar` 一致。
  - `packaging/hudi-cli-bundle` 的 artifactId 为 `hudi-cli-bundle_2.12`，与 Dockerfile 第 43 行 COPY 路径一致。
  - 已核实 `hudi-utilities/pom.xml` 与 `hudi-spark-datasource/pom.xml` 均不含任何 `flink` 依赖，因此两个 bundle 的 `-am` 依赖闭包不会引入 Flink 模块，裁剪后不会遗留未构建的依赖。
- 本修复不涉及正则替换第三方源文件，无需正则匹配验证。

## 潜在风险
- `-pl ... -am` 会缩小构建范围，仅产出 spark/cli bundle 及其依赖；若后续镜像还需要 Flink 相关产物，则需同步调整该命令。当前 Dockerfile 仅使用 spark/cli bundle，风险可控。
- 若上游 `release-1.2.1` 的模块目录或 artifactId 命名发生变动（当前版本未变动），`-pl` 路径可能失效；已按 `release-1.2.1` 实际仓库内容核实。
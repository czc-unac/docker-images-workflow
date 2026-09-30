# 修复摘要

## 修复的问题
Apache Druid 38.0.0 要求 Java 21，但镜像仍安装 Java 17，导致运行时 `UnsupportedClassVersionError`（容器启动/check 阶段失败）。

## 修改的文件
- `Bigdata/druid/38.0.0/24.03-lts-sp4/Dockerfile`: 将运行阶段的 `java-17-openjdk-headless` 改为 `java-21-openjdk-headless`。

## 修复逻辑
CI 分析报告因日志缺失无法定位根因，并提示需重点核查启动校验（check）阶段。经独立验证确认了版本相关的真实根因：

1. **上游要求已变更**：从上游 apache/druid 对应 tag `druid-38.0.0` 拉取 `pom.xml`，其中 `<java.version>21</java.version>`（`maven.compiler.release=21`）；官方镜像 `distribution/docker/Dockerfile` 在 38.0.0 使用 `ARG JDK_VERSION=21`，而 37.0.0 及更早版本均为 17。
2. **实际制品确认**：从华为云镜像站下载 `apache-druid-38.0.0-bin.tar.gz`（HTTP 200，已确认存在，排除下载 404 方向），解出 `extensions/druid-avro-extensions/druid-avro-extensions-38.0.0.jar` 中的 class 文件，major version = 65，即 Java 21 编译产物；Java 17 仅支持到 major 61，无法加载。
3. **结论**：38.0.0 二进制包内类文件为 Java 21 目标版本，而 Dockerfile 安装 Java 17，容器启动 `./bin/start-druid` 时会因 class 版本不兼容而失败，与报告"方向 2（check/启动阶段）"吻合。
4. **可用性确认**：openEuler 24.03-LTS-SP4 的 release 与 update 仓库均存在 `java-21-openjdk-headless` 包；仓库内其他镜像（如 `Base/gradle/9.7.1/24.03-lts-sp4`）已成功使用该包，升级安全。

修复仅将 Java 版本从 17 提升到 21，改动一行，属于版本升级所需的最小适配。

## 潜在风险
无。仅提升运行阶段 JDK 版本，Druid 38.0.0 本身以 Java 21 构建；Java 21 向后兼容既有依赖。`java-21-openjdk-headless` 在 24.03-LTS-SP4 仓库可用且已被仓库内其他镜像采用。其余文件（README.md、doc/image-info.yml、meta.yml）格式正确，无需改动。
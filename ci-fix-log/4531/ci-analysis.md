# CI 失败分析报告

## 基本信息
- PR: #4531 — 【自动升级】hudi容器镜像升级至1.2.1版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 模块依赖未构建
- 新模式症状关键词: Could not resolve dependencies, Could not find artifact, hudi-flink2.1.x, hudi-flink-client, BUILD FAILURE

## 根因分析

### 直接错误
```
#13 729.0 [ERROR] Failed to execute goal on project hudi-flink-client: Could not resolve dependencies for project org.apache.hudi:hudi-flink-client:jar:1.2.1
#13 729.0 [ERROR] dependency: org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 (compile)
#13 729.0 [ERROR] 	Could not find artifact org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 in Maven Central (https://repo.maven.apache.org/maven2)
#13 729.0 [ERROR] 	Could not find artifact org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 in confluent (https://packages.confluent.io/maven/)
#13 729.0 [ERROR] 	Could not find artifact org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 in jitpack.io (https://jitpack.io)
#13 729.0 [ERROR] 	Could not find artifact org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 in cloudera-repo-releases (https://repository.cloudera.com/artifactory/public/)
#13 729.0 [ERROR] 	org.apache.hudi:hudi-flink2.1.x:jar:1.2.1 was not found ... This failure was cached in the local repository
#13 729.0 [INFO] BUILD FAILURE
#13 729.0 [INFO] hudi-cli ........................................... SUCCESS [ 32.241 s]
#13 729.0 [INFO] hudi-flink-client .................................. FAILURE [  9.725 s]
#13 ERROR: process "/bin/sh -c mvn clean package -DskipTests -Dspark3.5 -Dscala-2.12" did not complete successfully: exit code: 1
Dockerfile:21
  21 | >>> RUN mvn clean package -DskipTests -Dspark3.5 -Dscala-2.12
```

### 根因定位
- 失败位置: `Bigdata/hudi/1.2.1/24.03-lts-sp4/Dockerfile:21`
- 失败原因: 该行 `mvn clean package -DskipTests -Dspark3.5 -Dscala-2.12` 以整仓 Reactor 方式构建 Hudi；`hudi-flink-client` 模块声明了对兄弟模块 `org.apache.hudi:hudi-flink2.1.x:1.2.1` 的编译期依赖，但该模块在当前 `-Dspark3.5 -Dscala-2.12` 的激活 profile 下未被纳入本次 Reactor（Reactor Summary 中不存在 `hudi-flink2.1.x` 条目，且其位于构建序列更靠后的 flink 分组），Maven 转而向 Maven Central / confluent / jitpack / cloudera 等远程仓库解析该内部制品，均不存在，最终 `Could not resolve dependencies` 导致 BUILD FAILURE。

### 与 PR 变更的关联
- 该失败与 PR 直接相关：PR 为新增文件 `Bigdata/hudi/1.2.1/24.03-lts-sp4/Dockerfile`，失败行即该文件第 21 行的 `mvn clean package` 命令。本次 PR 引入了 hudi 1.2.1 版本的构建逻辑，而 1.2.1 的模块结构使得在仅启用 spark3.5 / scala-2.12 的情况下执行全量 `package` 会触发 `hudi-flink-client` 对未构建 `hudi-flink2.1.x` 模块的依赖解析失败。
- README.md、doc/image-info.yml、meta.yml 的改动均为文档/元数据，不构成失败原因。

## 修复方向

### 方向 1（置信度: 高）
调整 Dockerfile 第 21 行的 Maven 构建命令，使构建范围仅覆盖最终需要的打包模块（本 Dockerfile 仅 COPY 了 `hudi-cli-bundle` 与 `hudi-spark3.5-bundle`），避免将 `hudi-flink-client` 等 flink 相关模块纳入构建：
- 通过 `-pl` 显式列出目标打包模块并使用 `-am` 构建其必要上游依赖；或
- 通过 Maven 参数将 flink 模块整体排除/跳过（如 `-Dflink...` 相关 profile 排除或模块排除），使构建不解析 `hudi-flink2.1.x` 内部依赖。
目标：让 `mvn clean package` 不再尝试解析未被构建的 flink 子模块制品。

### 方向 2（可选，置信度: 中）
若确需全量构建，则需显式将 `hudi-flink2.1.x` 纳入 Reactor（激活相应 flink profile，或去掉会跳过该模块的 profile 组合），使 Reactor 内可解析该内部依赖。此方向会显著增加构建耗时与产物体积，且与本 Dockerfile 实际使用产物不匹配，建议优先采用方向 1。

## 需要进一步确认的点
- 确认 Hudi 1.2.1 上游 `pom.xml` 中 `hudi-flink-client` 对 `hudi-flink2.1.x` 依赖的声明方式，以及 `hudi-flink2.1.x` 模块被纳入 Reactor 所需的 profile 名称/条件（用于确定是 `-pl`/`-am` 还是 profile 排除方案）。
- 确认本仓库中 `Bigdata/hudi/1.2.0/24.03-lts-sp4/Dockerfile` 当时成功构建所用的 mvn 命令形式，作为对齐参考（1.2.0 可通过说明其模块结构不同或命令不同）。
- 确认 CI 对 hudi 镜像的构建产物是否仅需要 `hudi-cli-bundle` 与 `hudi-spark3.5-bundle`，以验证“限定构建模块”方案不会遗漏运行期依赖。

## 修复验证要求
- 若采用 `-pl`/`-am` 方案：code-fixer 必须在提交前本地或 CI 中实际执行修改后的 `mvn` 命令，确认 Reactor 中不再出现 `hudi-flink-client` 对 `hudi-flink2.1.x` 的解析错误，且 `packaging/hudi-cli-bundle/target/hudi-cli-bundle_2.12-1.2.1.jar` 与 `packaging/hudi-spark-bundle/target/hudi-spark3.5-bundle_2.12-1.2.1.jar` 均成功生成（与第 31-32 行 COPY 路径一致）。
- 若采用 profile 排除方案：需验证排除后构建仍能产出上述两个 bundle jar，且未因排除 profile 导致 spark3.5 相关产物缺失。
- 因置信度为“高”，上述验证主要防止“构建范围限缩后产物缺失”的次生问题，不涉及正则 patch 外部源文件。

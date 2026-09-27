# CI 失败分析报告

## 基本信息
- PR: #4642 — 【自动升级】parquet容器镜像升级至1.18.1版本.
- 失败类型: dependency-error
- 置信度: 中
- 知识库匹配: 模式02（下载 URL 硬编码版本路径错误 / 软件包版本不存在；症状相近的还有模式38 Apache 404）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#7 [2/2] RUN curl -fSL -o parquet.tar.gz https://archive.apache.org/dist/parquet/apache-parquet-format-1.18.1/apache-parquet-format-1.18.1.tar.gz;     mkdir -p /usr/local/parquet &&     tar -zxf parquet.tar.gz -C /usr/local/parquet --strip-components=1 &&     rm -rf parquet.tar.gz
#7 0.070   % Total    % Received % Xferd ...
#7 0.070   0   196    0     0    0      0      0 --:--:-- --:--:-- --:--:--     0
#7 0.922 curl: (22) The requested URL returned error: 404
#7 0.927 tar (child): parquet.tar.gz: Cannot open: No such file or directory
#7 0.927 tar (child): Error is not recoverable: exiting now
#7 0.927 tar: Child returned status 2
#7 0.927 tar: Error is not recoverable: exiting now
#7 ERROR: process "... curl ... apache-parquet-format-${VERSION}/..." did not complete successfully: exit code: 2
------
Dockerfile:6
--------------------
   6 | >>> RUN curl -fSL -o parquet.tar.gz https://archive.apache.org/dist/parquet/apache-parquet-format-${VERSION}/apache-parquet-format-${VERSION}.tar.gz; \
   7 | >>>     mkdir -p /usr/local/parquet && \
   8 | >>>     tar -zxf parquet.tar.gz -C /usr/local/parquet --strip-components=1 && \
   9 | >>>     rm -rf parquet.tar.gz
ERROR: failed to solve: process "..." did not complete successfully: exit code: 2
```

### 根因定位
- 失败位置: `Bigdata/parquet/1.18.1/24.03-lts-sp4/Dockerfile:6`
- 失败原因: Dockerfile 中 `ARG VERSION=1.18.1` 拼出的下载地址 `https://archive.apache.org/dist/parquet/apache-parquet-format-1.18.1/apache-parquet-format-1.18.1.tar.gz` 返回 HTTP 404，`curl -fSL` 以 exit 22 结束，未生成 `parquet.tar.gz`，紧随其后的 `tar` 因文件不存在而报 `Error is not recoverable`，最终 `docker build` exit code 2。第一个错误即 404，后续 tar 报错均为其连带结果。
- 补充证据: `Bigdata/parquet/doc/image-info.yml` 中该镜像 `upstream.version_prefix: apache-parquet-format`、`backend: GitHub`，说明自动升级工具按 `apache-parquet-format` 前缀构造下载 URL。仓库内已有 `2.12.0` / `2.11.0` 等 parquet-format 条目，而本次新增为 `1.18.1`，版本号明显低于已有版本且可能并非 `apache-parquet-format` 的真实发布版本（`1.x` 更像 parquet-java/parquet-mr 的版本线）。

### 与 PR 变更的关联
直接相关。该 PR 新增文件 `Bigdata/parquet/1.18.1/24.03-lts-sp4/Dockerfile`（并同步更新 README.md、doc/image-info.yml、meta.yml），失败正是发生在这个新增 Dockerfile 的第 6 行下载步骤上。即本次 PR 引入的版本/URL 组合在 `archive.apache.org` 上不存在。

## 修复方向

### 方向 1（置信度: 中）
核对 `apache-parquet-format` 上游是否存在 `1.18.1` 这个版本。若不存在（很可能，因为格式库版本线为 2.x），说明自动升级取到了错误的上游版本号（疑似误取 parquet-java/parquet-mr 的 1.x），需要改用 `apache-parquet-format` 实际可用的版本，或修正 `doc/image-info.yml` 中的 `version_prefix`/上游来源，使升级工具取到正确的发布版本。

### 方向 2（置信度: 中）
若 `1.18.1` 确实存在，则说明是 URL 路径/文件名拼接格式错误（Apache archive 的目录名与产物名不一定都带 `apache-` 前缀）。需要以上游实际发布目录为准修正下载路径与文件名，而非继续使用 `${VERSION}` 统一拼接。

## 需要进一步确认的点
- `apache-parquet-format` 是否发布过 `1.18.1`：需访问 `https://archive.apache.org/dist/parquet/` 目录列表确认实际存在哪些版本目录（如 `apache-parquet-format-2.9.0/`、`parquet-...` 等命名）。
- 该 URL 的正确拼写规则（目录名、文件名是否带 `apache-` 前缀）需以上游 archive 实际内容为准。
- `1.18.1` 的来源：确认自动升级工具是依据 `apache-parquet-format`（格式库）还是 parquet-java（Java 实现）版本线生成的版本号；两者版本线不同，`doc/image-info.yml` 的 `version_prefix: apache-parquet-format` 是否为正确来源。
- Dockerfile 中是否存在其它隐患（如仅解压源码但 `ENTRYPOINT ["parquet","--version"]` 找不到可执行文件、缺少构建依赖），因下载步骤提前失败，尚未进入后续阶段，无法从当前日志判断。

## 修复验证要求
本次不涉及正则 patch 外部源文件，无需该方法验证。但 code-fixer 在提交前必须：
1. 从上游确认 `apache-parquet-format` 的目标版本与下载路径（建议以上游 archive/dist 目录实际列表为准），确保新 URL 可 200 下载。
2. 若将版本改回格式库版本线，需同步更新 `meta.yml`、`README.md`、`doc/image-info.yml` 中的版本与链接，保持元数据一致。

# CI 失败分析报告

## 基本信息
- PR: #4620 — 【自动升级】oceanbase容器镜像升级至5.0.1版本.
- 失败类型: build-error
- 置信度: 中
- 知识库匹配: 模式27（GitHub Release URL 404），与模式02（下载 URL 硬编码版本路径错误 / 软件包版本不存在）相关
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#8 [3/4] RUN ARCH=$([ "amd64" = "amd64" ] && echo "x86_64" || echo "aarch64") && \
    API_URL="https://api.github.com/repos/oceanbase/oceanbase/releases/tags/v5.0.1" && \
    CE_RPM=$(curl -sL ${API_URL} | grep -oP '"name":\s*"\K[^"]*oceanbase-ce-[0-9][^"]*\.el8\.'"${ARCH}"'\.rpm' | head -1) && \
    LIBS_RPM=$(...oceanbase-ce-libs...) && \
    DOWNLOAD_URL="https://github.com/oceanbase/oceanbase/releases/download/v5.0.1" && \
    curl -fSL -o oceanbase-ce.rpm "${DOWNLOAD_URL}/${CE_RPM}" && \
    curl -fSL -o oceanbase-ce-libs.rpm "${DOWNLOAD_URL}/${LIBS_RPM}"
#8 0.537   % Total    % Received % Xferd ...
#8 0.929 curl: (22) The requested URL returned error: 404
#8 ERROR: process "..." did not complete successfully: exit code: 22
```
日志显示下载阶段只收到 9 字节响应即返回 `curl: (22) ... 404`，随后 BuildKit 报
`ERROR: failed to solve: process ... did not complete successfully: exit code: 22`，
并定位到 `Dockerfile:15`（`Database/oceanbase/5.0.1/24.03-lts-sp4/Dockerfile` 的下载 RUN 指令）。
构建在 x86-64 runner（`ecs-build-docker-x86-hk`，`ARCH=x86_64`）上执行，属真实构建失败，非 infra 问题。

### 根因定位
- 失败位置: `Database/oceanbase/5.0.1/24.03-lts-sp4/Dockerfile:15`（`RUN ... curl -fSL -o oceanbase-ce.rpm "${DOWNLOAD_URL}/${CE_RPM}"` 步骤）
- 失败原因: 由变量拼装出的 RPM 下载 URL 返回 HTTP 404。最可能的情况是 `CE_RPM` / `LIBS_RPM`
  两个变量经 `grep -oP` 从 GitHub API 响应中提取时**匹配为空**，使下载 URL 退化为裸发布路径
  `${DOWNLOAD_URL}/`（仅 9 字节响应、随即 404）；即 `v5.0.1` release 在 GitHub 上不存在，
  或其 release 资产文件名不匹配脚本硬编码的 `oceanbase-ce-[0-9]*.el8.${ARCH}.rpm` 规则。

### 与 PR 变更的关联
直接相关。本 PR 新增 `Database/oceanbase/5.0.1/24.03-lts-sp4/Dockerfile`，
其中第 15–21 行的下载逻辑即为失败点。README.md、`doc/image-info.yml`、`meta.yml`
仅为版本登记，不参与构建，与本失败无关。失败由新增 Dockerfile 的下载步骤直接触发。

## 修复方向

### 方向 1（置信度: 中）
核对 `oceanbase/oceanbase` 仓库 `v5.0.1` release/tag 是否真实存在，以及其 release 资产的
实际文件名格式。若 tag 命名或资产命名与脚本假设不一致，需同步修正 Dockerfile 中的
`API_URL`/`DOWNLOAD_URL` 的 tag 形式与 `grep -oP` 资产名匹配规则（当前硬编码 `.el8.${ARCH}.rpm`）。
若 `grep` 匹配为空，应在脚本中显式校验 `CE_RPM`/`LIBS_RPM` 非空后再下载，避免以空变量拼出 404 URL。

### 方向 2（可选）
若 5.0.1 并未在 GitHub Releases 上提供 `.el8.x86_64/.aarch64` RPM 资产（上游改由自有站点或
其他命名/其他 el 版本发布），则应改用实际可用的制品来源或资产命名，而不是沿用 4.4.2 的下载模板。

## 需要进一步确认的点
- `https://api.github.com/repos/oceanbase/oceanbase/releases/tags/v5.0.1` 是否返回有效 release
  （若返回 `{"message":"Not Found"}`，则 tag `v5.0.1` 不存在或名称不同，如 `v5.0.1.0` 等）。
- 该 release 下实际资产文件名是否为 `oceanbase-ce-*.el8.x86_64.rpm` / `oceanbase-ce-libs-*.el8.x86_64.rpm`，
  还是 `.el9.`、`.oe` 等其他命名（影响 `grep -oP` 正则）。
- 现有历史的 oceanbase 4.x Dockerfile 是否使用同一模板并成功（用于对比确认是版本问题还是模板问题）。
- API 请求是否因未携带认证/限流导致返回不含 `"name"` 的错误响应（次要怀疑点）。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
本次修复若涉及调整 Dockerfile 中的 `grep -oP` 资产名正则，code-fixer 在提交前必须：
以 Dockerfile 中 `ARG VERSION=5.0.1` 为准，实际请求
`https://api.github.com/repos/oceanbase/oceanbase/releases/tags/v5.0.1`，
获取该 release 的资产真实文件名列表，确认新正则能匹配到 `oceanbase-ce` 与 `oceanbase-ce-libs`
的对应架构 RPM 名称，并确认拼出的下载 URL 可返回 2xx（而非 404）后再提交。
由于日志仅能证明“下载 URL 404”，无法区分“tag 不存在”与“正则不匹配”，
不得在未验证上述任一前提的情况下直接假定修复方向正确。

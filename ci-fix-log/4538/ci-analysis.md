# CI 失败分析报告

## 基本信息
- PR: #4538 — 【自动升级】oceanbase容器镜像升级至5.0.1版本.
- 失败类型: build-error
- 置信度: 中
- 知识库匹配: 模式27（GitHub Release URL 404，症状与模式02亦高度相关）
- 新模式标题: （非新模式）
- 新模式症状关键词: （非新模式）

## 根因分析

### 直接错误
```
#8 [3/4] RUN ARCH=$([ "amd64" = "amd64" ] && echo "x86_64" || echo "aarch64") && \
    API_URL="https://api.github.com/repos/oceanbase/oceanbase/releases/tags/v5.0.1" && \
    CE_RPM=$(curl -sL ${API_URL} | grep -oP '"name":\s*"\K[^"]*oceanbase-ce-[0-9][^"]*\.el8\."'"${ARCH}"'\.rpm' | head -1) && \
    LIBS_RPM=$(...) && \
    DOWNLOAD_URL="https://github.com/oceanbase/oceanbase/releases/download/v5.0.1" && \
    curl -fSL -o oceanbase-ce.rpm "${DOWNLOAD_URL}/${CE_RPM}" && \
    curl -fSL -o oceanbase-ce-libs.rpm "${DOWNLOAD_URL}/${LIBS_RPM}"
#8 0.342   % Total    % Received ...
#8 0.342   0     9    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
#8 0.645 curl: (22) The requested URL returned error: 404
#8 ERROR: process ... did not complete successfully: exit code: 22
```

### 根因定位
- 失败位置: `Database/oceanbase/5.0.1/24.03-lts-sp4/Dockerfile:20`（即第 15–21 行下载 RUN 步骤的第一个下载 curl：`curl -fSL -o oceanbase-ce.rpm`）
- 失败原因: 构建阶段从 `https://github.com/oceanbase/oceanbase/releases/download/v5.0.1/${CE_RPM}` 下载 `oceanbase-ce` RPM 时返回 HTTP 404（curl exit code 22）。日志中该 curl 仅接收 9 字节（即 GitHub 404 响应体 `Not Found`），说明构造出的下载 URL 指向了不存在的资产；由于使用 `&&` 串联，`oceanbase-ce-libs` 的下载未再执行。

### 与 PR 变更的关联
本 PR 新增了 `Database/oceanbase/5.0.1/24.03-lts-sp4/Dockerfile`，其中新引入了基于 GitHub Release API 动态解析并下载 `oceanbase-ce` / `oceanbase-ce-libs` RPM 的逻辑。CI 在 `[3/4]` 步执行该新增逻辑时 404，属于本次 PR 改动**直接触发**的构建失败，与其余 README.md / image-info.yml / meta.yml 文档性变更无关。

## 修复方向

### 方向 1（置信度: 中）
核对上游 `oceanbase/oceanbase` 仓库中 v5.0.1 release 的**真实 tag 名与资产文件名**，并据此修正 Dockerfile 中的 tag 模板与 grep 匹配规则：
- 确认 release tag 是否为纯 `v5.0.1`，还是带有类似 `_CE_BP`/`_CE` 的版本后缀（仓库中已有镜像版本命名为 `4.4.2_CE_BP3`、`4.3.5_CE_BP2_HF1`，提示 oceanbase 的 GitHub tag/资产命名可能并非纯语义化版本号）。
- 确认资产名中的 `el8` 是否为当前版本实际使用的后缀（可能为 `el9` 或其他），以及是否同时提供 `x86_64` 与 `aarch64` 资产。
- 若 grep 因 tag 不存在或资产命名不匹配而解析为空，`CE_RPM` 为空字符串时下载 URL 退化为 `.../v5.0.1/`，同样会产生 404。

### 方向 2（置信度: 中）
若上游确实尚未针对 openEuler/el8 提供 5.0.1 对应架构的 RPM 制品（release 尚未发布或未上传资产），则该版本暂时无法构建，应回退该自动升级或等待上游发布制品后再重新触发。

## 需要进一步确认的点
1. 直接请求 `https://api.github.com/repos/oceanbase/oceanbase/releases/tags/v5.0.1`，确认该 tag 是否存在；若存在，列出其 `assets[].name`，核对是否存在匹配 `.el8.<arch>.rpm` 的 `oceanbase-ce-*` 与 `oceanbase-ce-libs-*` 资产。
2. 对照本地仓库中既有的 oceanbase Dockerfile（如 `Database/oceanbase/4.4.2_CE_BP3/24.03-lts-sp4/Dockerfile`、`4.3.5_CE_BP3/...`）的下载逻辑，确认它们的 tag 命名与资产后缀规则，判定 5.0.1 应采用的正确 tag 模板。
3. 确认失败发生在 `oceanbase-ce`（第一个 curl）而非 `oceanbase-ce-libs`：日志中 `0 9` 字节后即 404，符合第一个下载 curl 失败、后续 curl 因 `&&` 未执行的时序。
4. 确认 5.0.1 是否同时提供 `x86_64`/`aarch64` 两种架构资产；若仅部分架构可用，需在 `meta.yml` 中通过 arch 约束限制构建架构（参考模式30/31）。

## 修复验证要求
（置信度为"中"，code-fixer 在提交修复前必须完成以下验证，不得直接假设某个方向正确）
1. 从上游 `oceanbase/oceanbase` 获取 v5.0.1 release 的实际信息（tag 名 + 完整资产列表），确认本次 404 的具体原因（tag 不存在 / 资产命名后缀不符 / 架构缺失）。
2. 用与 Dockerfile 中完全一致的 grep 正则（含 `.el8.` 与 `${ARCH}` 拼接）对获取到的资产列表做一次匹配验证，确认修复后 `CE_RPM`、`LIBS_RPM` 均非空。
3. 确认最终构造的 `DOWNLOAD_URL/${CE_RPM}` 与 `DOWNLOAD_URL/${LIBS_RPM}` 均可实际返回 200（可 HEAD/GET 校验），再提交修复。
4. 若最终采用方向 2（回退版本），需确认回退不会破坏 README.md / image-info.yml / meta.yml 与新增 Dockerfile 的一致性。

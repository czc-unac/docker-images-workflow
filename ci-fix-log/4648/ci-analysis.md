# CI 失败分析报告

## 基本信息
- PR: #4648 — 【自动升级】starrocks容器镜像升级至4.1.3版本.
- 失败类型: infra-error
- 置信度: 中
- 知识库匹配: 新模式
- 新模式标题: 下载脚本429限流
- 新模式症状关键词: curl: (22), 429, Too Many Requests, build.sh, chmod: cannot access

## 根因分析

### 直接错误
```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
curl: (22) The requested URL returned error: 429
chmod: cannot access 'build.sh': No such file or directory
/tmp/jenkins7009743826141710098.sh: line 21: ./build.sh: No such file or directory
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: Jenkins 执行 shell 阶段（`/tmp/jenkins7009743826141710098.sh` 第 21 行），发生在下载 `build.sh` 的 `curl` 步骤
- 失败原因: CI 编排脚本通过 `curl` 下载 `build.sh` 时，服务端返回 **HTTP 429（Too Many Requests，请求被限流）**，导致 `build.sh` 未被落盘；随后 `chmod build.sh` 与 `./build.sh` 依次报 "No such file or directory"，job 被标记为失败。

### 与 PR 变更的关联
本 PR 仅新增/修改以下文件，均不涉及 CI 编排脚本或 `build.sh` 的获取逻辑：
- `Bigdata/starrocks/4.1.3/24.03-lts-sp4/Dockerfile`（新增，共 18 行）
- `Bigdata/starrocks/README.md`、`Bigdata/starrocks/doc/image-info.yml`、`Bigdata/starrocks/meta.yml`（新增 4.1.3 条目）

失败发生在 Docker 构建**之前**的 `build.sh` 下载阶段，属于 CI 基础设施的下载限流问题，与 PR 代码改动**无直接关联**。

## 修复方向

### 方向 1（置信度: 中）
重新触发（retry）该构建。HTTP 429 是服务端瞬时限流，通常重试即可恢复；本次 PR 未改动任何构建逻辑，重跑是最优先验证手段。

### 方向 2（置信度: 低）
若重试后仍持续返回 429，则是下载源对 CI 构建机 IP 的持续限流，需要 CI/基础设施维护方调整 `build.sh` 的获取方式（如更换下载源、增加退避重试或缓存）。

## 需要进一步确认的点
1. 完整日志缺失：当前仅提供 "Error lines" 与 "Build tail"，需确认 `build.sh` 的下载 URL 与触发 429 的具体服务端（gitee/atomgit/内部制品库）。
2. 是否 aarch64 侧 job 同样出现 429，以判断是全局限流还是单节点问题。
3. PR 是否带有 `ci_failed` 标签且日志确为失败 job（本日志末尾为 `Finished: FAILURE`，非 SUCCESS，故属于真实失败 job，但内容已被截断/摘要化）。

## 修复验证要求
不适用（本次修复方向不涉及对第三方/上游源文件的正则 patch）。
